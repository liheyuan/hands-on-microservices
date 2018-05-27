# Mockito 单元测试打桩神器

## 单元测试

软件测试是软件质量保证的关健环节，代表了需求、设计和编码的最终检查。

![软件测试金字塔](./test.jpg "软件测试金字塔")

如上图所示，测试金字塔将测试分为三类
* 单元测试: 由开发者自行编写，应当覆盖80%以上的场景。对微服务架构而言，主要是在单个微服务内部，对复杂业务逻辑，编写单元测试。
* 集成测试: 由测试人员编写，强调系统整体联动，多偏向业务可用性验证。如下单流程是否畅通，库存扣减是否成功。它的覆盖场景一般之占10%。
* 功能测试: 一些由单元测试、集成测试不好做的，通过功能测试完成，一般来说，这类不好自动化的，需要手动进行测试。由于测试成功很高，这类一般只覆盖5%的场景。

测试金字塔也向我们揭示了一个实时：单元测试是整个测试环节的根基，如果单元测试做不好，上层的集成测试、功能测试都会无从谈起。

遗憾的是，多数开发者都不具备编写单元测试的良好习惯，甚至缺乏编写单元的动力。

除了缺乏软件质量保障的意识外，"嫌麻烦"也是这类开发者面对单元测试的口头禅。

本节将介绍Mockito，这是一个单元测试的利器。Mockito的出现，让我们可以更加轻松地编写单元测试。

在介绍Mockito之前，先来解释下，我们为什么不推荐使用Spring Boot启动单元测试框架。

实际上，Spring Boot本身是提供了单元测试框架的，可以在JUnit中通过注解的配置，启动一个Spring上下文环境，并支持自动注入等功能，如果你感兴趣，可以参考[这篇文档](http://www.baeldung.com/spring-boot-testing)。

在实际工作中，我也尝试过上述方法，但效果却并不太好，主要原因是：
* 启动Spring Boot环境速度很慢，至少要3秒，而一般的单元测试都是毫秒级别。
* 依赖管理需要手动声明，随着业务不断升级，经常忘记维护单元测试中的依赖，导致单元测试无法正常执行。

基于上述原因，我强烈不推荐在单元测试中启动Spring Boot环境。

对于服务之间存在依赖关系的场景，建议直接使用Mockito的打桩(Mock)进行。

希望在仔细的阅读本节后，你也会爱上单元测试:-)

## Mockito

在软件测试中，Mock指的是效仿、模仿。Mockito就是为了解决测试中的Mock问题而诞生的，它可以很好的解决单元测试中，由于不同类耦合而带来的难以测试的问题。

还是以上面Spring Boot环境为例子。假设我们要测试A类，而类A又调用了B类和C类。此时可能有两种选择：

1. 手动构造B和C。
1. 通过启动Spring环境，自动地注入B和C。

现在有了Mockito后，我们有了另外的思路：无需构造B和C，而是通过Mockito，"Mock"出B和C(构造符合接口但没有实现的类)，由于我们要测试的是A类中的逻辑，只要检查A调用B和C的时机、次数、参数是否正确，就可以了。

我们通过一个例子，来说明mockito的用法。

首先是ServiceA和它的实现:
```java
package com.coder4.lmsia.abc.server.service.intf;

/**
 * @author coder4
 */
public interface ServiceA {

    int methodA(int a, int b);

}
```

```java
package com.coder4.lmsia.abc.server.service.impl;

import com.coder4.lmsia.abc.server.service.intf.ServiceA;
import com.coder4.lmsia.abc.server.service.intf.ServiceB;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * @author coder4
 */
@Service
public class ServiceAImpl implements ServiceA {

    @Autowired
    private ServiceB serviceB;

    @Override
    public int methodA(int a, int b) {
        if (a <= 10 && b <= 10) {
            return a + b;
        } else {
            return serviceB.methodB(a, b);
        }
    }
}
```

然后是服务B和它的实现:
```java
package com.coder4.lmsia.abc.server.service.intf;

/**
 * @author coder4
 */
public interface ServiceB {

    int methodB(int a, int b);

}
```

```java
package com.coder4.lmsia.abc.server.service.impl;

import com.coder4.lmsia.abc.server.service.intf.ServiceB;
import org.springframework.stereotype.Service;

/**
 * @author coder4
 */
@Service
public class ServiceBImpl implements ServiceB {

    @Override
    public int methodB(int a, int b) {
        return a * b;
    }
}
```

我们总结一下功能：

* 在服务A中，若参数a和b都小于10，则返回求和结果，否则交给服务B处理。
* 在服务B中，直接返回参数a和b的乘积结果。

在编写单元测试前，先要引用对应的包，lmabc-server/build.gradle：
```grovvy
dependencies {
    compile project(':lmsia-abc-common')
    ...
    testCompile 'junit:junit:4.12'
    testCompile 'org.mockito:mockito-all:1.9.5'
}
```

这里要指出的是，mockito本身还是需要单元测试框架才能运行的，我们这里用的是最常见的JUnit。

然后看一下单元测试
```java
package com.coder4.lmsia.abc.server;

import com.coder4.lmsia.abc.server.service.impl.ServiceAImpl;
import com.coder4.lmsia.abc.server.service.intf.ServiceA;
import com.coder4.lmsia.abc.server.service.intf.ServiceB;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;
import org.mockito.Mockito;
import org.mockito.internal.util.reflection.Whitebox;

import static org.hamcrest.CoreMatchers.is;

/**
 * @author coder4
 */
public class ServiceATest {

    private ServiceA serviceA;

    private ServiceB serviceB;

    @Before
    public void before() {
        serviceA = new ServiceAImpl();
        serviceB = Mockito.mock(ServiceB.class);
        Whitebox.setInternalState(serviceA, "serviceB", serviceB);
    }

    @Test
    public void testBelow10() {
        Assert.assertThat(serviceA.methodA(1, 1), is(2));

        Mockito.verifyZeroInteractions(serviceB);

    }

    @Test
    public void testAbove10() {
        serviceA.methodA(100, 1);

        Mockito.verify(serviceB).methodB(100, 1);

    }

}
```

我们分步解释一下：
* before: 初始化ServiceA，因为我们要测试这个，所以必须手动初始化。而ServiceB我们在A的测试并不关注，直接Mock一个，并通过Whitebox注入到服务A中。
* testBelow10: 前面服务实现已经介绍过，当参数a和b都小于10的场景，是在ServiceA中直接求和。所以这里我们验证两个的和，然后验证下是不是没有"碰过"服务B(verifyZeroInteractions)
* testAbove10: 当任何一个参数大于10时候，实际会走服务B。所以我们验证下是否调用了服务B，且参数恰好是传给A的就好。

怎么样，有了Mockito后，测试是不是变得有趣起来了:-)

[官方文档](http://static.javadoc.io/org.mockito/mockito-core/2.18.3/org/mockito/Mockito.html)中提供了更多有趣的例子，等待你的发掘。
