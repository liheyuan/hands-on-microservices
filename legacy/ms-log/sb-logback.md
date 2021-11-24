# Spring Boot配置Logback及HTTP日志

系统上先后，需要进行一系列的运维、监控工作，可能还需要排查业务故障和系统问题。

服务已经上线了，不能像本地开发的一样“”打断点调试“，此时，日志的作用就非常重要了。

与其他服务框架类似，Spring Boot也默认集成了日志系统，默认采用Logback日志类库。

Logback是Log4j的作者开发的另一款日志类库，与其他同类竞品相比，它的优势有：
* 更高的性能，官方说比Log4j快10倍以上
* 原生兼容slf4j
* 支持多环境配置、自动切换、压缩等高级功能

在本节的前半部分，我们将讨论如何在Spring Boot中如何使用Logback。本节的后半部分，我们看一下如何在Spring Boot中启用HTTP访问日志(内嵌的Tomcat日志)。

## Spring Boot中配置Logback

在Spring Boot中配置Logback只需要两步：
* 确认类路中含有logback，这一般是通过其他starter自动带上的，例如spring-boot-starter-web
* 定义配置文件:logback-spring.xml

我们来看一下配置好的文件:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- For console -->
    <appender name="ConsoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <charset>UTF-8</charset>
            <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] [%thread] [%logger] [tr=%mdc{TRACE_ID:-0}] %msg %n</Pattern>
        </encoder>
    </appender>

    <!-- For file with daily rollover -->
    <appender name="ServerFileAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <encoder>
            <charset>UTF-8</charset>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] [%thread] [%logger] [tr=%X{TRACE_ID:-0}] %msg %n</pattern>
        </encoder>

        <file>/app/logs/lmsia-abc.log</file>

        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- daily rollover with gz -->
            <fileNamePattern>/app/logs/lmsia-abc.%d{yyyy_MM_dd}.log.gz</fileNamePattern>
            <!-- keep 30 days' max -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>

    <!-- console only if local active -->
    <springProfile name="local">
        <root level="INFO">
            <appender-ref ref="ConsoleAppender"/>
        </root>
    </springProfile>
    <!-- file only if test or online active -->
    <springProfile name="test,online">
        <root level="INFO">
            <appender-ref ref="ServerFileAppender"/>
        </root>
    </springProfile>

</configuration>

```

如上所示，我们的配置中包含了2个Appender(可理解为两种日志输出方法)：
* ConsoleAppender: 直接输出到命令行
* ServerFileAppender: 输出到/app/logs/lmsia-abc.log文件中，并且:按天自动切换文件、并做gz压缩、最多保留30天。

上述切换、压缩、30天仅通过几行就搞定了，可见logback的强大之处！

在下面，我们通过对不同profile的判断，可以让不同的Appender生效。
当我们在本地执行时，默认是local的profile，此时我们只运行ConsoleAppender，即直接输出到命令行，方便调试。
当在服务器执行时，如测试环境test和生产环境online，我们只启用ServerFileAppender。因为此时没有人会看stdout的输出，都是通过看文件的方式来看日志的。

最后，我们简单看一下日志格式，即Pattern:
```
%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] [%thread] [%logger] [tr=%X{TRACE_ID:-0}] %msg %n
```

几个部分分别表示:
* 日期，如2018-06-07 18:30:23.124
* 日志级别，INFO, ERROR等
* 日志线程，如果有名字会优先用名字，没有用线程ID
* Logger名字，一般是类名
* TraceId，追踪信息，下一节将介绍它
* 消息体及换行，如果有异常及异常栈，会自动输出在后面

在代码中使用，也是非常简单：

```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private Logger LOG = LoggerFactory.getLogger(getClass());

LOG.info("Test");
```

## 配置Tomcat日志

Spring Boot默认内置了Tomcat服务器，从而实现了真正的"开箱即用"，如何开启Tomcat的HTTP访问日志呢？

一般有两种方法:
* 在yaml中配置
* 通过代码实现

其中yaml中配置的方案最简单，但每个项目都要配置一次，非常麻烦，网上资料很多，这里不做介绍了。

我们重点看一下第二种方案，我们可以将它抽成一个包，别的项目引用这个包时候，自动启用HTTP访问日志。

```java
import org.apache.catalina.valves.AccessLogValve;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
import org.springframework.boot.context.embedded.ConfigurableEmbeddedServletContainer;
import org.springframework.boot.context.embedded.EmbeddedServletContainerCustomizer;
import org.springframework.boot.context.embedded.tomcat.TomcatEmbeddedServletContainerFactory;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
@ConditionalOnWebApplication
public class TomcatAccessLogConfiguration
        extends WebMvcConfigurerAdapter implements EmbeddedServletContainerCustomizer {

    private Logger LOG = LoggerFactory.getLogger(getClass());

    @Override
    public void customize(ConfigurableEmbeddedServletContainer container) {
        if (container instanceof TomcatEmbeddedServletContainerFactory) {
            TomcatEmbeddedServletContainerFactory factory = (TomcatEmbeddedServletContainerFactory) container;
            AccessLogValve accessLogValve = new AccessLogValve();
            accessLogValve.setEnabled(true);
            accessLogValve.setDirectory("/app/logs/");
            accessLogValve.setPattern("common");
            accessLogValve.setSuffix(".log");
            factory.addContextValves(accessLogValve);
        } else {
            LOG.error("This customizer does not support your configured container!");
        }
    }
}

```

如上所示:
* 当启用Web时，自动激活这个自动配置
* 使用默认的日志格式common
* 路径在/app/logs下，后缀是.log

当然不要忘记加一个spring.factories
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.coder4.lmsia.commons.http.configuration.TomcatAccessLogConfiguration
```

这样，当别的Spring Boot项目引用这个库时，就会自动启用HTTP日志了。

这个HTTP日志也是支持按天滚动，只不过不支持压缩，如果你想对其进行更多定制，推荐直接阅读Tomcat的相关源代码。
