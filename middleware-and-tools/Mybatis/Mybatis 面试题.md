### **`#{}` 和 `${}` 的区别<font color="red">（重点）</font>**

**(1) 基本概念**

- `#{}`: **安全地将动态参数绑定到 SQL 语句中**，而不是直接将参数插入到 SQL 字符串里。
- `${}` 主要是用于**字符串替换**，适用于需要动态替换 SQL 片段的场景。

**(2) 安全性**

- `#{}` 可以**有效防止 SQL 注入攻击**。
-  ${} 不能防止 SQL 注入攻击， 直接插入的内容需要手动验证安全性。

**(3) 应用场景**

- `#{}`： 用于动态参数绑定，适合大多数场景。

- `${}`：适用于需要动态替换 SQL 片段的特殊场景

**(4) `${}` 特定的应用场景**

虽然 `${}` 可能导致 SQL 注入，但是**某些特定场景（动态替换 SQL 片段)下只能使用 `${}`** 。特殊场景包括：

- **动态拼接数据库对象名称**（表名、列名等）。如果需要动态拼接表名或列名，使用 `#` 是无法实现的，因为 `#` 占位符会将参数作为字符串处理，导致 SQL 无法正确执行。

  ```java
  @Select("select * from user where ${column} = #{value}")
  User findByColumn(@Param("column") String column, @Param("value") String value);
  ```

-  **动态拼接排序字段**。在动态排序场景中，排序字段需要直接拼接到 SQL 中，使用 `$` 是必要的。

  ```java
  @Select("SELECT * FROM users ORDER BY ${sortField} ${sortOrder}")
  List<User> findAll(@Param("sortField") String sortField, @Param("sortOrder") String sortOrder);
  ```

- **动态拼接函数或表达式**

  ```sql
  @Select("SELECT ${calculationExpression} AS result FROM data_table")
  Double calculate(@Param("calculationExpression") String calculationExpression);
  ```

**(5) #{} 的安全原理**

在将 SQL 语句发送到数据库前，MyBatis 会通过 MySQL 语句预编译阶段中 `PreparedStatement` 的 `?` 占位符机制替代参数，并在执行时通过 `setXxx` 方法将参数安全地填充到占位符中，从而避免直接拼接 SQL 字符串。

> SQL 语句在 MySQL 执行过程为：语法解析 -> 预编译 -> 查询优化 -> 执行

**使用占位符后，SQL 被分为两部分**：

- **固定部分**：SQL 的语法和逻辑，比如 `SELECT * FROM user WHERE name = ?`。
- **动态部分**：占位符对应的参数值。<font color="red">应用程序后续通过 `setXxx` 方法将值单独传递给数据库驱动</font>。

占位符替代拼接的过程如下：

```sql
# 1. SQL 预编译阶段
# 数据库只看到 ?，并将其作为参数位置标记
# 数据库编译执行计划时只处理 SQL 的固定部分，不关心 ? 的具体值。
SELECT * FROM user WHERE name = ?

# 2. SQL 执行阶段（参数绑定阶段）
# 应用程序通过 setXxx 方法将值传递给数据库驱动。
# 参数值被单独处理，作为数据直接传入数据库，不会干扰 SQL 的语法。
preparedStatement.setString(1, "Alice");
```

**占位符本质上是数据库和应用程序的协议约定**。数据库从不直接将参数值替换到 SQL 字符串中，而是将值与 SQL 的逻辑分开。通过这种方式，可以保证参数值不会破坏 SQL 的结构或注入恶意语句。

```sql
# 假定一个 SQL 注入攻击
preparedStatement.setString(1, "Alice'; DROP TABLE user;--");

# 数据库对其进行处理，转换成以下 SQL 语句
# SQL 注入攻击无效，因为分号和注释语法只是普通字符串
SELECT * FROM user WHERE name = 'Alice''; DROP TABLE user;--'
```



### 怎样防止 SQL 注入？

**(1) 什么是 SQL 注入**

**SQL 注入**（SQL Injection）是指攻击者通过向应用程序的输入框中插入恶意 SQL 代码，进而篡改或操控数据库查询，以达到窃取、篡改或删除数据等恶意目的的攻击方式。

**(2) 常见防止方式**

- **使用预编译语句（Prepared Statements）**：预编译语句将 SQL 查询与参数分离，使得输入的参数不会被当作 SQL 代码执行，从而避免了注入攻击。

- **输入验证和过滤**：在接收用户输入时，应该进行严格的验证和过滤，确保输入的数据格式合法，且不包含 SQL 特殊字符（如 `;`、`'`、`--`、`/*` 等）



### **Mybatis 怎么实现一对多查询？**

- **手动指定映射关系**：使用 `<resultMap>` 标签来指定如何将查询结果映射为 Java 对象，并处理一对多的关系。 但是，这个要求`User` 类中有一个 `List<Order>` 类型的 `orders` 属性。

  ```java
  public class User{
      private Long userId;
      private String userName;
      private List<Order> orders;
  
      // getters and setters
  }
  ```

  然后，在 mapper.xml 文件中，`resultMap` 通过 `collection` 标签将多个 `Order` 对象映射到 `User` 对象的 `orders` 属性中。

  ```xml
  <mapper namespace="com.example.mapper.UserMapper">
  
      <!-- 处理一对多关系 -->
      <resultMap id="userResultMap" type="com.example.User">
          <id property="id" column="userId"/>
          <result property="name" column="userName"/>
          <!-- 一对多关系映射 -->
          <collection property="orders" ofType="com.example.Order">
              <id property="id" column="orderId"/>
              <result property="orderNumber" column="orderNumber"/>
          </collection>
      </resultMap>
  
      <!-- 查询用户及其所有订单 -->
      <select id="selectUserWithOrders" resultMap="userResultMap">
          SELECT u.id AS userId, u.name AS userName, o.id AS orderId, o.order_number AS orderNumber
          FROM user u
          LEFT JOIN orders o ON u.id = o.user_id
          WHERE u.id = #{userId}
      </select>
  
  </mapper>
  
  ```

- **创建 DTO(Data Transfer Object)类**：

  - 定义 DTO 类，封装查询结果

    ```java
    public class UserOrderDTO {
        private Long userId;
        private String userName;
        private Long orderId;
        private String orderNumber;
    
        // getters and setters
    }
    ```

  - 在 MyBatis 的映射文件中，我们可以使用一个查询，返回多个字段，并且将结果映射到 `UserOrderDTO` 类中：

    ```xml
    <select id="selectUserOrders" resultType="com.example.dto.UserOrderDTO">
        SELECT u.id AS userId, u.name AS userName, o.id AS orderId, o.order_number AS orderNumber
        FROM user u
        LEFT JOIN orders o ON u.id = o.user_id
        WHERE u.id = #{userId}
    </select>
    ```

[MyBatis实现多表查询_mybatis两表联查条件查询-CSDN博客](https://blog.csdn.net/Mq_sir/article/details/116400731)



### Mybatis 支持的关联查询？

MyBatis 不仅可以执行一对一、一对多的关联查询，还可以执行多对一，多对多的关联查询。

- 多对一查询其实就是多个一对一查询。
- 多对多查询也相当于多个一对多查询。



### xml 映射文件中常见的标签

**(1) 执行 SQL 语句的基本标签**

这些标签主要用于执行 SQL 语句，如查询、插入、更新、删除等。

- **`<select>`**：执行查询操作，通常用于获取数据。
- **`<insert>`**：执行插入操作，将数据插入到数据库。
- **`<update>`**：执行更新操作，修改数据库中的数据。
- **`<delete>`**：执行删除操作，从数据库中删除数据。

**(2) 动态 sql 标签**

动态 SQL 标签用于根据条件动态生成 SQL 语句。这些标签在 SQL 查询中非常常见，特别是在需要灵活查询时。

- **`<where>`**：用于自动去除 SQL 中不必要的 `AND` 或 `OR` 前缀。
- **`<if>`**：用于根据条件是否成立动态生成 SQL 片段。
- **`<set>`**: 动态生成 `UPDATE` 语句中的 `SET` 子句 的一个标签
- **`<choose>`、`<when>` 和 `<otherwise>`**：类似于 `switch-case`，用于多条件选择。
- **`<foreach>`**：用于遍历集合，动态生成 SQL 语句中的多个值，常用于批量插入或查询。
- **`<trim>`**：用于去除 SQL 中多余的部分，如去掉多余的 `AND` 或 `OR`。
- **`<bind>`**：用于在 SQL 中定义并绑定变量，避免 SQL 注入或复杂的条件计算。

**(3) SQL 片段标签**

SQL 片段标签用于定义和重用 SQL 片段，避免代码重复。

- **`<sql>`**：定义可重用的 SQL 片段，可以在多个查询中复用。
- **`<include>`**：用于引入 SQL 片段，重用 `<sql>` 标签定义的内容。

**(4) 其他标签**

- **`<resultMap>`**：用于定义如何将查询结果映射到 Java 对象。

- **`<parameterMap>`**：用于定义如何将传入的参数映射到 SQL 查询语句中。



### Dao 接口原理

- DAO（Data Access Object）接口的原理基于 **动态代理机制**。MyBatis 使用 Java 的动态代理为 DAO 接口创建实现类，这些实现类在运行时自动生成并负责执行 SQL 语句。



### Dao 接口中的方法可以重载吗？

在 MyBatis 中，DAO 接口的方法名对应 XML 配置中的 `id`，而 XML 配置的 `id` 必须唯一。

因此，Mybatis 的 Dao 接口**可以有多个重载方法**，但是多个方法对应的映射必须只有一个，否则启动会报错。



### xml 配置文件中的 id 是否能够重复？

在 MyBatis 中，`id` 的唯一性取决于是否使用了 `namespace`：

- **使用 `namespace`**：不同的 XML 映射文件中，`id` 可以重复。
- **未使用 `namespace`**：所有映射文件的 `id` 必须全局唯一。





### 为什么 MyBatis 被称为“半 ORM”框架

- **MyBatis**：MyBatis 提供了一定程度的对象关系映射功能，例如将查询结果映射为 Java 对象。但是，它没有完全的自动化映射功能，**开发者需要手动定义 SQL 和对象之间的关系**。特别是对于关联关系的处理（如一对一、一对多、多对多等），需要开发者自己编写复杂的 SQL 查询和映射配置。

- **传统 ORM**：传统的 ORM 框架如 Hibernate 通常具备高度自动化的映射能力，支持复杂的对象关系（如实体类的继承、聚合、级联操作等），并通过注解或 XML 配置来定义这些关系。**框架会根据定义好的映射关系自动生成合适的 SQL**，并在数据库和对象之间自动转换数据。



### 参考资料

[MyBatis 常见面试题37道-包含答案_mybatis面试-CSDN博客](https://blog.csdn.net/Wyxl990/article/details/136049772)

[MyBatis常见面试题总结 | JavaGuide](https://javaguide.cn/system-design/framework/mybatis/mybatis-interview.html#为什么说-mybatis-是半自动-orm-映射工具-它与全自动的区别在哪里)