# Spring Boot集成SQL数据库2

## Spring Boot 集成 MyBatis操作MySQL

MyBatis是一款半自动的ORM框架。由于某国内大厂的广泛使用，MyBatis在国内非常火热(在国外其热度不如Hibernate)。

首先还是集成依赖：

```groovy
implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'
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
  configuration:
    map-underscore-to-camel-case: true
  type-aliases-package: com.coder4.homs.demo.server.mybatis.dataobject
```

不难发现，数据库链接的定义复用了jdbc的那一套，MyBatis的定义分3行，如下：

- configuration：开启驼峰规则转化

- type-aliases-package：mapper文件存放的包名

更多MyBatis的配置选项可以参考[这里]([mybatis-spring-boot-autoconfigure – Introduction](https://mybatis.org/spring-boot-starter/mybatis-spring-boot-autoconfigure/))

接着，我们定义Mapper，在MyBatis中，Mapper相当于前面手写的Repository，定义如下：

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

    @Insert("INSERT INTO users(name) VALUES(#{name})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    long create(UserDO user);

    @Select("SELECT * FROM users WHERE id = #{id}")
    UserDO getUser(@Param("id") Long id);

    @Select("SELECT * FROM users WHERE name = #{name}")
    UserDO getUserByName(@Param("name") String name);

}
```

你可能会奇怪：这不是接口(interface)么，并没有实现？

是的，通过定义@Repository和@Mapper，MyBatis会通过运行时的切面注入，帮我们自动实现，具体执行的SQL和映射，会读取@Select、@Options等注解中的配置。

经过上述介绍，你可以发现：

MyBatis可以直接通过注解的方式快速访问数据库，(相对于JDBC的)精简了大量无用代码。

同时，MyBatis依然需要指定运行的SQL语句，这与JDBC的方式是一致的。虽然有些繁琐，但可以保证性能可控。

如果你在网上搜索"MyBatis Spring集成"，会找到大量xml配置的用法。

在一些老项目中，xml是标准的集成方式。在这种配置方式下，配置繁琐、代码量大，即使借助"MyBatisX"等插件，也依然较为复杂。

因此，除非你要维护遗留的老项目代码，我都建议你使用(本文中)注解式集成MyBatis。

## Spring Boot集成 JPA 操作MySQL

JPA的全称是Java Persistence API，即持久化访问规范API。

Spring也提供了集成JPA的方案，称为 Spring Data JPA，其底层是通过Hibernate的JPA来实现的。

首先集成依赖：

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
implementation 'mysql:mysql-connector-java:8.0.20'
```

与前面类似，不再重复介绍。

接着是配置：

```yaml
# jdbc demo
spring.datasource:
  url: jdbc:mysql://127.0.0.1:3306/homs_demo?useUnicode=true&characterEncoding=UTF-8&useSSL=false
  username: HomsDemo
  password: 123456
  hikari:
    minimumIdle: 10
    maximumPoolSize: 100


# jpa demo
spring.jpa:
  database-platform: org.hibernate.dialect.MySQL8Dialect
  hibernate.ddl-auto: validate
```

在MySQL连接上，我们依然复用了Spring DataSource的配置。

jpa侧的配置为：

- database-platform：设置使用MySQL8语法

- hibernate.ddl-auto：只校验表，不回主动更新数据表的结构

接着，我们来定义实体(Entity)：

```java
@Entity
@Data
@Table(name = "users")
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    // @Column(name = "name")
    private String name;

    public User toUser() {
        User user = new User();
        user.setId(id);
        user.setName(name);
        return user;
    }

}
```

这里我们将UserEntity与表"users"做了关联。

接下来是Repository：

```java
@Repository
public interface UserJPARepository extends CrudRepository<UserEntity, Long> {

    Collection<UserEntity> findByName(String name);

}
```

我们继承了CrudRepository，他会自动生成针对UserEntity的CRUD操作。

此外，我们还定义了1个额外函数：

- findByName，通过隐士语法规则，让JPA自动帮我们生成对应SQL

从直观感受上，JPA比MyBatis更加“高级” -- 一些简单的SQL都不用写了。

但天下真的有免费的馅饼么？我们先卖个关子。

## JMJ应该选哪个

经过这两节的介绍，你已经掌握了JDBC、MyBatis、JPA三种操作数据库的方式。

在实战中，究竟要选哪个呢？

从易用性的角度来评估，我们可以得出结论：JPA > MyBatis > JDBC

那么从性能的角度来看呢？

我们使用wrk做了(get-by-id接口的)简单压测，结论如下：

|         | 读QPS |
| ------- | ---- |
| JDBC    | 457  |
| MyBatis | 445  |
| JPA     | 114  |

这里，你会惊讶的发现：

- JDBC和MyBatis的性能差别不大，在5%以内

- JPA(Hibernate)的性能，居然只有其余两种方式的1/3 

如此差的性能，真的让人百思不得其解，我尝试打印了SQL和执行耗时，并没有发现什么异常。

更进一步的，我们尝试用指定SQL的方式，替换了自动生成的接口，如下

```java
@Repository
public interface UserJPARepository extends CrudRepository<UserEntity, Long> {

    @Query(value = "SELECT * FROM users WHERE id = :id", nativeQuery = true)
    Optional<UserEntity> findByIdFast(@Param("id") long id);

}
```

这次的压测结果是：447，性能基本和JDBC持平了。但是这种NativeSQL的用法并没有使用自动生成SQL的功能，没有发挥Hibernate本来的功效。

所以，我们认为，锅在于Hibernate自动生成SQL的逻辑耗时过大。

当然，Hibernate也不是一无是处，针对多层关联，建模复杂的场景，使用Entity做映射，会更加方便。

让我们回到前面的问题上：JMJ应该选哪个？

- 如果对性能有极致要求，建议JDBC或者MyBatis。

- 如果建模场景复杂，嵌套密集，且对性能要求不高，可以选用Hibernate。
