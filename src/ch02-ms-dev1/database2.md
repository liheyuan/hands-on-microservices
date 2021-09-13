# Spring Boot集成SQL数据库2

## Spring Boot 集成 MyBatis访问MySQL

MyBatis是一款半自动的ORM框架。由于某国内大厂的广泛使用，MyBatis在国内非常火热(在国外其热度不如Hibernate)。

首先还是集成依赖：

```groovy
implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter'
implementation 'mysql:mysql-connector-java:8.0.20'
```

套路与jdbc类似，但starter并不是官方的了，而是mybatis自己做的starter，感兴趣的可以来[这里](https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter/2.2.0)看下具体组成(会有惊喜)。

接下来是yaml配置环节：

```yaml
spring.datasource:
  url: jdbc:mysql://127.0.0.1:3306/homs_demo?useUnicode=true&characterEncoding=UTF-8&useSSL=false
  username: HomsDemo
  password: 123456
  hikari:
    minimumIdle: 10
    maximumPoolSize: 100

# mybatis extra
mybatis:
  config-location: classpath:mybatis-config.xml
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.coder4.homs.demo.server.mybatis.dataobject
```

不难发现，数据库链接的定义复用了jdbc的那一套，MyBatis的定义分3行，如下：

- config-location：主配置文件在resource/mybatis-config.xml下

- mapper-locations：mapper文件路径在resource/mapper下

- type-aliases-package：mapper文件存放的包名

看下mybatis-config.xml配置：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>

</configuration>
```

这里只配置了命名的驼峰转化规则，更多配置可以参考[官方文档]([mybatis &#x2013; MyBatis 3 | 配置](https://mybatis.org/mybatis-3/zh/configuration.html))。

然后我们定义Mapper，在MyBatis中，Mapper相当于前面手写的Repository，定义如下：

```java
package com.coder4.homs.demo.server.mybatis.mapper;

import com.coder4.homs.demo.server.mybatis.dataobject.UserDO;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.springframework.stereotype.Repository;

/**
 * <p>
 *  Mapper 接口
 * </p>
 *
 * @author author
 * @since 2021-09-09
 */
@Repository
@Mapper
public interface UserMapper {

    long create(UserDO user);

    UserDO getUser(@Param("id") Long id);

    UserDO getUserByName(@Param("name") String name);

}
```

你可能会奇怪：这不是接口(interface)么，并没有实现？

是的，通过定义@Repository和@Mapper，MyBatis会通过运行时的切面注入，帮我们自动实现！

那么具体实现时执行什么SQL呢？需要你在对应xml中编写，

mapper/UserMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >

<mapper namespace="com.coder4.homs.demo.server.mybatis.mapper.UserMapper">
    <sql id="FIELDS">
        id, name
    </sql>

    <insert id="create" useGeneratedKeys="true" keyProperty="id" parameterType="com.coder4.homs.demo.server.mybatis.dataobject.UserDO">
        INSERT INTO users (
        name
        ) VALUES (
        #{name}
        )
    </insert>
    <select id="getUser"
            parameterType="Long"
            resultType="com.coder4.homs.demo.server.mybatis.dataobject.UserDO">
        SELECT
            <include refid="FIELDS"></include>
        FROM users
        WHERE id = #{id}
    </select>
    <select id="getUserByName"
            parameterType="String"
            resultType="com.coder4.homs.demo.server.mybatis.dataobject.UserDO">
        SELECT
        <include refid="FIELDS"></include>
        FROM users
        WHERE name = #{name}
    </select>
</mapper>
```

如上所述，我们在xml中配置了每一个方法对应的SQL语句，部分参数是动态的，但都很好理解。

无须多言，MyBatis的使用非常繁琐，虽然省略了部分Java代码，但复杂的XML配置和奇怪的语法，都增加了学习与维护门槛。除非你的项目中需要海量SQL语句，否则引入MyBatis是得不偿失的。

为了减少代码量，社区开发了多款自动生成MyBatis"骨架代码"的插件，例如“Free MyBatis Plugin”、“MyBatisX”等。他们可以部分缓解上述问题，但生成的代码较难维护。

MyBatis也是有优点的：可以直接操作SQL语句，性能可控。

## Spring Boot集成 JPA 访问MySQL


