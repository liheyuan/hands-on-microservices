# Spring Boot整合配置中心

上一小节中，我们探讨了如何利用gerrit搭建配置中心的版本仓库。

现在，我们探讨如何在Spring Boot的框架中整合配置中心。

## 开发lmsia-cfg4j库，实现配置项的自动注入

与之前的Cache等功能类似，我们在多个微服务中，轻松地使用配置中心，所以将相关功能提取但独立的项目中。你可以在这里查看[lmsia-cfg4j](https://github.com/liheyuan/lmsia-cfg4j)的源代码。

如前文描述，我们使用cfg4j来辅助实现配置中心的功能。

cfg4j提供了默认的"Binding"方式，来绑定配置项到类中，但使用起来较为繁琐。

许多Spring的功能都是通过注解来实现的，非常方便。我们的配置项也可以用注解来实现，首先定义一个注解接口:

```java
package com.coder4.lmsia.cfg4j;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @author coder4
 */
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Cfg4jValue {

    String value() default "";
}

```

如上所示，之后希望可以使用类似"@Cfg4jValue"的方式，将配置项注解到对应字段中。

有了注解接口，如何实现自动注解呢，传统的方式需要使用动态代理来完成，在这里我们采用Spring提供的BeanPostProcessor来完成：
```java
package com.coder4.lmsia.cfg4j;

import org.cfg4j.provider.ConfigurationProvider;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.aop.support.AopUtils;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.core.Ordered;
import org.springframework.util.ReflectionUtils;

import java.lang.reflect.Field;
import java.util.NoSuchElementException;

/**
 * @author coder4
 */
public class Cfg4jValueProcessor implements BeanPostProcessor, Ordered {

    private Logger LOG = LoggerFactory.getLogger(getClass());

    @Autowired
    private ConfigurationProvider configurationProvider;

    // 初始化前注入
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

        final Class targetClass = AopUtils.getTargetClass(bean);
        ReflectionUtils.doWithFields(targetClass, field -> process(bean, targetClass, field), field -> {
            return field.isAnnotationPresent(Cfg4jValue.class);
        });
        return bean;
    }

    private void process(final Object bean, Class<?> targetClass, final Field field) {
        // Get injected field name
        Cfg4jValue valueAnnotation = field.getDeclaredAnnotation(Cfg4jValue.class);
        String fieldName = getPropName(valueAnnotation, field.getName());
        // inject for some support type
        fieldSetWithSupport(bean, field, fieldName);
    }

    private void fieldSetWithSupport(Object bean, Field field, String key) {
        Class type = field.getType();
        field.setAccessible(true);
        try {
            if (int.class == type || Integer.class == type) {
                field.set(bean, configurationProvider.getProperty(key, Integer.class));
            } else if (boolean.class == type || Boolean.class == type) {
                field.set(bean, configurationProvider.getProperty(key, Boolean.class));
            } else if (String.class == type) {
                field.set(bean, configurationProvider.getProperty(key, String.class));
            } else if (long.class == type || Long.class == type) {
                field.set(bean, configurationProvider.getProperty(key, Long.class));
            } else {
                LOG.error("not support cfj4j value inject type");
                throw new RuntimeException("not supported cfg4jValue type");
            }
        } catch (IllegalAccessException e) {
            LOG.error("exception during field set", e);
            throw new RuntimeException(e);
        } catch (NoSuchElementException e) {
            LOG.error("config missing key, please check");
            throw new RuntimeException(e);
        }
    }

    public static String getPropName(Cfg4jValue annotation, String defaultName) {
        String key = annotation.value();
        if (key == null || key.isEmpty()) {
            key = defaultName;
        }
        return key;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws
            BeansException {
        return bean;
    }

    @Override
    public int getOrder() {
        return HIGHEST_PRECEDENCE;
    }
}
```

如上所示，Cfg4jValueProcessor完成了以下功能:
* 自动查找所有@Cfg4jValue的注解
* 对有上述注解的字段，根据字段名从Cfg4j的数据源(ConfigurationProvider)中读取配置项
* 若有配置项，完成类型转换并注入到对应字段中。这里目前只支持int, long, string这三种类型。

ConfigurationProvider是cfg4j的数据源，如前文所述，我们希望它自动从gerrit来读取。

为此，实现一个自动配置如下:
```java
package com.coder4.lmsia.cfg4j.configuration;

import com.coder4.lmsia.cfg4j.Cfg4jValueProcessor;
import org.cfg4j.provider.ConfigurationProvider;
import org.cfg4j.provider.ConfigurationProviderBuilder;
import org.cfg4j.source.ConfigurationSource;
import org.cfg4j.source.context.environment.Environment;
import org.cfg4j.source.context.environment.ImmutableEnvironment;
import org.cfg4j.source.context.filesprovider.ConfigFilesProvider;
import org.cfg4j.source.git.GitConfigurationSourceBuilder;
import org.cfg4j.source.reload.ReloadStrategy;
import org.cfg4j.source.reload.strategy.PeriodicalReloadStrategy;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Service;

import java.nio.file.Paths;
import java.util.Arrays;
import java.util.concurrent.TimeUnit;

/**
 * @author coder4
 */
@Configuration
@ConditionalOnProperty("msName")
public class Cfg4jGitConfiguration {

    @Value("${msName}")
    private String msName;

    // May Change this
    private static String CONFIG_GIT_HOST = "10.1.64.72";

    // May Change this
    private static String CONFIG_GIT_REPO = "http://" + CONFIG_GIT_HOST + ":9002/lmsia-config.git";

    // May Change this
    private static String branch = "master";

    private static int RELOAD_SECS = 60;

    @Bean
    public ConfigurationProvider configurationProvider() {
        ConfigFilesProvider configFilesProvider = () -> Arrays.asList(Paths.get(msName + "/config.properties"));
        ConfigurationSource source = new GitConfigurationSourceBuilder()
                .withRepositoryURI(CONFIG_GIT_REPO)
                .withConfigFilesProvider(configFilesProvider)
                .build();

        Environment environment = new ImmutableEnvironment(branch);

        ReloadStrategy reloadStrategy = new PeriodicalReloadStrategy(RELOAD_SECS, TimeUnit.SECONDS);

        return new ConfigurationProviderBuilder()
                .withConfigurationSource(source)
                .withEnvironment(environment)
                .withReloadStrategy(reloadStrategy)
                .build();
    }

    @Bean
    public Cfg4jValueProcessor createCfg4jValueProcessor() {
        return new Cfg4jValueProcessor();
    }

}
```

上述完成了如下功能:
* 从Git仓库拉去lmsia-config项目（即前文用于微服务配置的仓库）
* 定义缓存时间为60秒
* 配置文件具体路径为/项目名/config.properties(与前一节的目录结构相对应)
* 顺便配置刚才编写的Cfg4jValueProcessor，让配置可以自动注入到对应的地方上。

当然，为了让上述自动注解生效，不要忘记配置spring.factories
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.coder4.lmsia.cfg4j.configuration.Cfg4jGitConfiguration
```

至于项目名，可以通过微服务自身的application.yaml指定，若不制定将不会启动这个自动配置

## 使用

有了lmsia-cfg4j后，如何在微服务中自动注入配置项目呢？

首先我们需要准备好配置，例如lmsia-config/lmsia-abc的config.properties中定义
```
key=value
enable=false
```

接着，在微服务的application.yaml中定义项目名称
```yaml
msName: lmsia-abc
```

最后一步，在代码中，使用注解：
```java
package com.coder4.lmsia.abc.server.configuration;

import com.coder4.lmsia.cfg4j.Cfg4jValue;
import lombok.Data;
import org.springframework.stereotype.Service;

/**
 * @author coder4
 */
@Service
@Data
public class TestConfig {

    @Cfg4jValue
    private String key;

    @Cfg4jValue
    private boolean enable;

}
```

如上所示，是不是非常简单！

你可以启动自己的微服务项目，测试上述配置项是否被如期的注入进来。

在lmsia-cfg4j中，有默认60秒的缓存，你也可以修改lmsia-abc的配置，等待60秒，再观察新的配置是否生效。

## 拓展与思考

1. 我们介绍的的配置中心架构中，实际采用的是拉默认来获取最新配置。当微服务及其副本数量众多的时候，可能会对gerrit服务器造成巨大压力。有什么好的方法可以改进这一点么?
2. 如果需要结合profile实现测试、线上环境使用不同的配置，lmsia-cfg4j项目要如何进行修改呢？ 
