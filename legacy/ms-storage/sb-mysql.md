# Spring Boot整合MySQL

经过上一节的讨论，相信你已经有了一套可运维的MySQL服务器了，接下来的两节，我们来讨论如何在Spring Boot中整合MySQL。

在Spring Boot中整合MySQL有很多方式，常见的有:
* Spring Jdbc Template直接集成
* Spring Data JPA集成
* Hibernate集成
* MyBatis集成

使用过Spring框架开发的同学，可能对后两种比较熟悉。但是这两种方法过于重量级，本书将专注于前两种，其中：
* Jdbc Template需要直接编写SQL语句，更加接近数据库底层，开发效率低、性能高。
* Spring Data JPA可以自动生成部分参数、解析结果的语句，开发效率高，性能低一些。
上述两种方法各有优略，大家可以根据实际情况作出选择。

## 数据源配置

无论是选用哪种集成方式，数据源的集成都是必不可少的。

为了提升性能，一般会使用数据库连接池，我们采用Spring Boot默认的tomcat连接池，只需要如下依赖配置即可生效：
```
    compile 'org.springframework.boot:spring-boot-starter-jdbc'
    compile 'mysql:mysql-connector-java:5.1.9'
```

接下来我们看一下数据源的配置，在application.yaml中添加:
```yaml
spring.datasource:
  url: jdbc:mysql://mysql/lmsia_abc?rewriteBatchedStatements=true
  username: lmsia
  password: pass
  testOnBorrow: true
  validationQuery: SELECT 1
  tomcat:
    max-active: 500
```

如上所示，除了基本的url和用户名、密码外，还设定了一系列额外参数，这些都是生产环境建议设置的，解释一下：
* testOnBorrow / validationQuery: 从连接池取出连接后，先检查是否可用。这主要是解决长时间空闲情况下MySQL Server的[Gone Away问题(]https://dev.mysql.com/doc/refman/8.0/en/gone-away.html)
* tomcat.max-active: 连接池最大连接数设定为500，默认的100在高并发场景下可能不够。
* rewriteBatchedStatements: 只有设置为true，才会默认启用batch模式，可提升批量写入的性能。

添加了上述配置后，Spring Boot会自动生成DataSource以及NamedParameterJdbcTemplate。我们可以通过后者直接操作数据库。

## 通过JDBCTemplate操作数据库

在操作数据库前，我们先来看一下数据表结构：
```sql
CREATE TABLE IF NOT EXISTS `user` (
    `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(256) NOT NULL,
    `createdTime` BIGINT(20) NOT NULL,
    `updatedTime` BIGINT(20) NOT NULL,
    PRIMARY KEY (`id`)
);
```

在微服务开发中，一般会将表的一行映射成一个实体:
```java
import lombok.Data;

@Data
public class User {

    private int id;

    private String name;

    private long createdTime;

    private long updatedTime;

}
```

其中上面的@Data是lombok的注解，用于帮我们自动生成getter和setter，感兴趣的同学可以看这lombok官方文档](https://projectlombok.org/)，这里不再详述。

来看一下数据库操作，我们将其封装在了Repository中:
```java
@Repository
public class UserRepositoryImpl implements UserRepository {

    protected Logger LOG = LoggerFactory.getLogger(getClass());

    @Autowired
    protected NamedParameterJdbcTemplate db;

    private RowMapper<User> ROW_MAPPER = new BeanPropertyRowMapper(User.class);

    @Override
    public void add(User user) {
        String sql = "INSERT INTO `user`(`name`, `createdTime`, `updatedTime`) VALUES " +
                "(:name, :createdTime, :updatedTime)";
        SqlParameterSource param = new BeanPropertySqlParameterSource(user);
        KeyHolder keyHolder = new GeneratedKeyHolder();
        db.update(sql, param, keyHolder);
        LOG.info("insert succ, id = {}", keyHolder.getKey().longValue());
    }

    @Override
    public Optional<User> getUserById(int id) {
        String sql = "SELECT * FROM `user` WHERE `id` = :id";
        SqlParameterSource param = new MapSqlParameterSource("id", id);
        try {
            return Optional.ofNullable(db.queryForObject(sql, param, ROW_MAPPER));
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
}
```

解读一下上面的代码:
* 通过Autowired自动注入NamedParameterJdbcTemplate
* JdbcTemplate上执行update和query来完成插入或查询
* 查询参数通过SqlParameterSource传入，返回值的对象映射通过RowMapper完成。

## 两个数据源

在上面的数据源配置、数据库操作中，都存在一个假设：只有一个数据源。

如果一个微服务要同时依赖多个数据库，需要做如下事情:
* 配置不同的数据源，建议不要采用默认的spring.datasource前缀，这主要是为了避免@Autowired时命名冲突。
* 为多个数据源手动声明Configuration，包含多组DataSource和JdbcTemplate

例如我们现在要添加2个数据库的数据源，那么配置文件要变成:
```yaml

db1.datasource:
  url: jdbc:mysql://mysql/db1?rewriteBatchedStatements=true
  username: db1
  password: pass
  testOnBorrow: true
  validationQuery: SELECT 1
  tomcat:
    max-active: 500

db2.datasource:
  url: jdbc:mysql://mysql/db2?rewriteBatchedStatements=true```
  username: db2
  password: pass
  testOnBorrow: true
  validationQuery: SELECT 1
  tomcat:
    max-active: 500
```

由于不采用默认的spring.datasource前缀了，Spring Boot默认不会激活自动配置，需要手动编写：
```java
@Configuration
@EnableTransactionManagement
public class DataSourceConfiguration {

    @Bean(name = "db1JdbcTemplate")
    @Primary
    public NamedParameterJdbcTemplate initDb1JdbcTemplate(
            @Autowired @Qualifier("db1DataSource") DataSource dataSource) {
        return new NamedParameterJdbcTemplate(dataSource);
    }

    @Bean
    @Primary
    @ConfigurationProperties(prefix = "db1.datasource")
    public DataSource db1DataSource() {
        return DataSourceBuilder.create().build();
    }


    @Bean(name = "db2JdbcTemplate")
    public NamedParameterJdbcTemplate initDb2JdbcTemplate(
            @Autowired @Qualifier("db2DataSource") DataSource dataSource) {
        return new NamedParameterJdbcTemplate(dataSource);
    }

    @Bean
    @ConfigurationProperties(prefix = "db2.datasource")
    public DataSource db2DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public PlatformTransactionManager txManager() {
        return new DataSourceTransactionManager(tutorClockinWriterDataSource());
    }
}
```

简单说明一下：
* 根据自定义的前缀生成对应DataSource。
* 根据DataSource生成对应的NamedParameterJdbcTemplate。
* 因为要生成两组Datasource和NamedParameterJdbcTemplate，所以有一组要设置为@Primary，这是Spring Boot的要求。

在使用时，因为有两个NamedParameterJdbcTemplate了，所以要补充一下名字以做区分，如下：
```java
@Autowired
@Qualifier("db1")
protected NamedParameterJdbcTemplate db1;

@Autowired
@Qualifier("db2")
protected NamedParameterJdbcTemplate db2;
```

区分了不同的NamedParameterJdbcTemplate后，其余的数据库操作和一个Datasource时是完全相同的，这里不再赘述。

# 通过JPA操纵数据库

前面提到了，除了JdbcTemplate外，还可以使用JPA来操作数据库。

由于spring-boot-starer-data-jpa显示以依赖了spring-boot-starter-jdbc，所以我们可以直接替换依赖:
```grovvy
    compile 'org.springframework.boot:spring-boot-starter-data-jpa'
```

这一步替换，将不会影响DataSource、JdbcTemplate的自动注入，JPA也是需要Datasource和JdbcTemplate才能正常完成工作的。

Spring Boot JPA的默认实现是通过Hibernete完成的（JPA只是一套接口，Hibernete是接口的一种实现）。

jpa需要在yaml中添加一些特殊配置:
```yaml
spring.jpa.properties.hibernate.dialect: org.hibernate.dialect.MySQL5InnoDBDialect
spring.jpa.hibernate.ddl-auto: validate
spring.jpa.hibernate.naming.physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
```

其中:
* hibernate.dialect让Hibernete可以更高效的生成sql
* ddl-auto设置为validate，不自动创建表但是会验证表与实体是否符合
* naming.physical-strategy表字段名映射为驼峰命名

针对要操作的实体，需要做一些特殊注解，以让JPA可以关联到对应的表上，为了对比说明，我们单独创建了一个对象UserForJpa:
```java
import lombok.Data;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;
import javax.persistence.Temporal;
import javax.persistence.TemporalType;

/**
 * @author coder4
 */
@Data
@Entity
@Table(name = "user")
public class UserForJpa {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(nullable = false)
    private String name;

    @Column(name = "createdTime", nullable = false, updatable = false)
    private long createdTime;

    @Column(name = "updatedTime", nullable = false)
    private long updatedTime;

}
```

说明一下：
* 实体通过@Entity标注
* @Table关联实体和表
* @Id和@GeneratedValue完成自增主键的声明
* @Column是普通列的声明，可设置是否nullable以及是否可更新

在数据库操作中，我们可以完全让JPA帮我们生成sql，如下:
```java
@Repository
public interface UserJpaRepository extends JpaRepository<UserForJpa, Integer> {


}
```

是的，你没有看错，我们不需要编写任何方法，就能自动获得save(), findOne(), findAll(), count(), delete()等接口，具体可以参见JpaRepository的源代码。

我们看一下调用方式:
```java
userJpaRepository.findOne(userId);

userJpaRepository.save(user);
```

JpaRepository提供的都是较为基础的操作，有事无法完全满足我们的需求。我们可以自行定义sql，如下：
```java
    @Query(
            value = "SELECT * FROM `user` ORDER BY `id` DESC LIMIT 1",
            nativeQuery = true)
    UserForJpa findLatestUser();
```

如上所示，我们通过@Query注解实现了通过指定SQL查找最新注册的用户。

## 小结

在本小节中，我们首先介绍了Sping Boot中MySQL数据源的配置，随后，介绍了如何配置多个数据源并手动注入DataSource、JdbcTemplate。

接下来，我们介绍了两种数据库操作方法:
* JdbcTemplate更接近数据库底层，需要编写较多代码，性能较好
* Spring JPA Data可以自动生成部分代码，开发效率高，性能稍差，且对POJO具有一定的侵入性

上述两种方法各有优劣，大家可以根据实际需求进行选择。

## 拓展阅读

1. Tomcat数据库连接池的详细配置参数可以参考[官方参数文档](https://tomcat.apache.org/tomcat-8.0-doc/jdbc-pool.html#Common_Attributes)
1. Spring JPA Data更详细的用法可以参考[Spring JPA Data官方文档](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
1. Spring JDBC更详细的用法可以参考[Spring JDBC官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html)

