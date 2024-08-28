## MyBatis概述

MyBatis 是一款优秀的持久层 ORM（Object-Relational Mapping）框架，主要用于将 Java 对象与数据库中的记录相映射。与其他 ORM 框架如 JPA、Hibernate 相比，MyBatis 的显著特点在于它对 SQL 语句的高度定制化支持。通过 MyBatis，开发者可以直接使用简单的 XML 或注解来配置 SQL 语句和映射规则，大大减少了传统 JDBC 编程中繁琐的代码，如手动设置参数和获取结果集的操作。

MyBatis 的核心理念是将 Java 的 POJO（普通的 Java 对象）与数据库表中的记录进行映射。这一映射过程可以通过 XML 文件进行配置，也可以通过注解方式来实现。JavaBean、POJO 和 Entity 都是 MyBatis 中常见的概念，分别对应 Java 对象、数据库表中的记录及持久化对象。

### MyBatis执行流程

MyBatis 的执行流程大致如下：
1. **Configuration加载**：首先，MyBatis 通过 `mybatis-config.xml` 加载全局配置文件，这个文件包含数据库连接信息、Mapper 配置等。
2. **SessionFactory创建**：通过 `Configuration` 创建一个 `SqlSessionFactory`，用于生成 `SqlSession` 实例。
3. **SqlSession创建与使用**：`SqlSession` 是 MyBatis 操作数据库的核心接口。它负责执行已加载的 SQL 语句 (`MappedStatement`) 并管理事务 (`Transaction`)。
4. **MappedStatement加载与执行**：每个 SQL 语句在配置文件中都有对应的 `MappedStatement`，通过 `SqlSession` 加载并执行这些语句，从而实现数据库操作。

### MyBatis的核心配置与使用

MyBatis 的核心在于 SQL 语句的配置与执行。每个 `statement` 都对应一条 SQL 语句，通过 `<insert>`、`<delete>`、`<update>`、`<select>` 标签进行配置。

#### `id`属性
`id` 是每个 `statement` 的唯一标识，确保 `statement` 可以被成功定位和执行。

#### `resultType`属性
仅用于 `<select>` 标签，用于指定 SQL 返回结果的类型。例如，查询结果可以指定为 `com.example.entity.Person` 类型。

#### `parameterType`属性
用于指定传入 SQL 语句的参数类型。如果不指定，MyBatis 会自动判断该类型。

#### `resultMap`属性
用于解决复杂的结果集与 Java 对象之间的映射关系，例如数据库字段与实体类属性不一致时，通过 `resultMap` 手动配置。

#### 动态SQL

MyBatis 通过 `if`、`where`、`foreach` 等标签来实现动态 SQL 的生成，从而提高 SQL 语句的灵活性。

- **`foreach`** 标签可用于处理集合或数组参数，适用于动态生成 `IN` 子句等。
- **`set`** 标签用于自动处理 `update` 语句中的 `set` 子句，避免手动处理最后一个逗号的问题。

### MyBatis多表映射

MyBatis 还支持多表关联查询，可以通过 `<association>` 和 `<collection>` 标签实现多对一、一对多的映射关系。

- **`<association>`** 标签用于一对一的关系映射，例如一个书籍对应一个作者。
- **`<collection>`** 标签用于一对多的关系映射，例如一个作者对应多本书籍。


## 案例
### 1. `application.yml` 配置文件

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydatabase
    username: root
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver
  mybatis:
    mapper-locations: classpath:mappers/*.xml
    type-aliases-package: com.example.entity
```

### 2. 实体类

```java
@Data
public class Person {
    private Integer id;
    private String name;
    private Integer age;
    private Address address; // 一对一映射
}
```

```java
@Data
public class Address {
    private Integer id;
    private String street;
    private String city;
}
```

### 3. Mapper XML 文件

#### `src/main/resources/mappers/PersonMapper.xml`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<mapper namespace="com.example.mapper.PersonMapper">
    <resultMap id="personResultMap" type="com.example.entity.Person">
        <id property="id" column="id" />
        <result property="name" column="name" />
        <result property="age" column="age" />
        <association property="address" javaType="com.example.entity.Address">
            <id property="id" column="address_id" />
            <result property="street" column="address_street" />
            <result property="city" column="address_city" />
        </association>
    </resultMap>

    <select id="findPersonById" resultMap="personResultMap">
        SELECT p.id, p.name, p.age, a.id as address_id, a.street as address_street, a.city as address_city
        FROM person p
        LEFT JOIN address a ON p.address_id = a.id
        WHERE p.id = #{id}
    </select>

    <insert id="insertPerson" parameterType="com.example.entity.Person">
        INSERT INTO person (name, age, address_id) VALUES (#{name}, #{age}, #{address.id})
    </insert>

    <update id="updatePerson" parameterType="com.example.entity.Person">
        UPDATE person
        SET name = #{name}, age = #{age}, address_id = #{address.id}
        WHERE id = #{id}
    </update>

    <delete id="deletePerson" parameterType="Integer">
        DELETE FROM person WHERE id = #{id}
    </delete>
</mapper>
```

### 4. Mapper 接口

```java
@Mapper
public interface PersonMapper {
    Person findPersonById(@Param("id") Integer id);
    void insertPerson(Person person);
    void updatePerson(Person person);
    void deletePerson(@Param("id") Integer id);
}
```

### 5. 服务层

```java
@Service
public class PersonService {
    
    @Autowired
    private PersonMapper personMapper;

    public Person getPersonById(Integer id) {
        return personMapper.findPersonById(id);
    }

    public void addPerson(Person person) {
        personMapper.insertPerson(person);
    }

    public void updatePerson(Person person) {
        personMapper.updatePerson(person);
    }

    public void deletePerson(Integer id) {
        personMapper.deletePerson(id);
    }
}
```

### 6. 控制层

```java
@RestController
@RequestMapping("/persons")
public class PersonController {

    @Autowired
    private PersonService personService;

    @GetMapping("/{id}")
    public Person getPerson(@PathVariable Integer id) {
        return personService.getPersonById(id);
    }

    @PostMapping
    public void addPerson(@RequestBody Person person) {
        personService.addPerson(person);
    }

    @PutMapping
    public void updatePerson(@RequestBody Person person) {
        personService.updatePerson(person);
    }

    @DeleteMapping("/{id}")
    public void deletePerson(@PathVariable Integer id) {
        personService.deletePerson(id);
    }
}
```

### 7. 动态 SQL 示例

假设我们想要通过动态 SQL 条件查询人员信息，可以使用 MyBatis 的 `<if>` 标签：

```xml
<select id="findPersonsByConditions" resultMap="personResultMap">
    SELECT p.id, p.name, p.age, a.id as address_id, a.street as address_street, a.city as address_city
    FROM person p
    LEFT JOIN address a ON p.address_id = a.id
    WHERE 1=1
    <if test="name != null">
        AND p.name = #{name}
    </if>
    <if test="age != null">
        AND p.age = #{age}
    </if>
</select>
```


## 总结

MyBatis 作为一款轻量级的 ORM 框架，提供了高度灵活的 SQL 语句配置能力，并且通过动态 SQL 和多表映射功能，极大地方便了复杂业务逻辑的实现。在现代 Java 企业应用中，MyBatis 是一个不可或缺的工具，尤其是在需要对 SQL 进行细粒度控制的场景中，它展现出了强大的优势。