基于MicroMeter实现自定义监控项

## MicroMeter简介

xx

## JMX自定义基本监控项

引入pom

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-jmx</artifactId>
            <version>1.8.7</version>
        </dependency>
```

开发代码

```java
package com.coder4.homs.micrometer.web;


import com.coder4.homs.micrometer.web.data.UserVO;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.PostConstruct;

@RestController
public class UserController {

    @Autowired
    private MeterRegistry meterRegistry;

    private Counter COUNTER_GET_USER;

    @PostConstruct
    public void init() {
        COUNTER_GET_USER = meterRegistry.counter("app_requests_method_count", "method", "UserController.getUser");
    }

    @GetMapping(path = "/users/{id}")
    public UserVO getUser(@PathVariable int id) {
        UserVO user = new UserVO();
        user.setId(id);
        user.setName(String.format("user_%d", id));

        COUNTER_GET_USER.increment();
        return user;
    }

}
```

测试

执行2次

```shell
curl "127.0.0.1:8080/users/1"
{"id":1,"name":"user_1"}
```

jconsole

![f](./jconsole1.png)

f
