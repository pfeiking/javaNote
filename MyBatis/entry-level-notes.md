[toc]

# 1 MyBatis的开发流程

Mybatis官网：https://mybatis.org/mybatis-3/zh/index.html

1. 引入MyBatis依赖
2. 创建核心配置文件
3. 创建entity
4. 创建Mapper映射文件
5. 初始化SessionFactory
6. 利用SqlSession对象操作数据

## 1.1 引入MyBatis依赖（Maven）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.java</groupId>
    <artifactId>mybatis</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.mybatis/mybatis -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.5.1</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.47</version>
        </dependency>
    </dependencies>
</project>
```

## 1.2 创建核心配置文件

在/src/resources/文件夹下创建mybatis-config.xml，并进行配置

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <!-- goods_id -> goodsId 驼峰命名转换-->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
    <!-- 通过切换default可以灵活切换数据库-->
    <environments default="dev">
        <environment id="dev"> //环境配置
            <transactionManager type="JDBC"></transactionManager>//采用什么事务对数据库进行管理
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/babytun?useUnicode=true&amp;characterEncoding=UTF-8"/>
                <property name="username" value="root"/>
                <property name="password" value="wwe61846"/>
            </dataSource>
        </environment>

        <environment id="prd">
            <!-- 采用JDBC方式对数据库事务进行commit/rollback -->
            <transactionManager type="JDBC"></transactionManager>
            <!-- 采用连接池的方式管理数据库 -->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://192.168.1.155:3306/babytun?useUnicode=true&amp;characterEncoding=UTF-8"/>
                <property name="username" value="root"/>
                <property name="password" value="wwe61846"/>
            </dataSource>
        </environment>
    </environments>
</configuration>
```

当将`<environment id="dev">`修改为`<environment id="prd">`之后将会使用id为“prd的环境，更换数据库等。



## 1.5 初始化SessionFactory

### SqlSessionFactory

- 创建MyBatis的核心对象
- 用于初始化MyBatis创建SqlSession对象
- 保证SqlSessionFactory在应用中**全局唯一**

### SqlSession

- 是MyBatis操作数据库的核心对象
- 使用JDBC方式与数据库交互
- 提供了CRUD对应方法

```java
@Test
public void testSqlSessionFactory() throws IOException {
  // 利用Reader加载classpath下的mybatis-config.xml核心配置文件
  Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
  // 初始化SqlSessionFactory对象，同时解析mybatis-config.xml文件
  SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
  System.out.println("SessionFactory加载成功");
  SqlSession sqlSession = null;
  try {
    // 创建SqlSession对象，SqlSession是JDBC的拓展类，用于与数据库交互
    sqlSession = sqlSessionFactory.openSession();	
    // 创建数据库连接（测试用），正常情况下是由MyBatis创建的，正常使用了MyBatis就不用其他java.sql的包了。
    Connection conn = sqlSession.getConnection();
    System.out.println(conn);
  }catch(Exception e){
    e.printStackTrace();
  }finally{
    // 如果type=“POOLED”，代表使用连接池，close则是将连接回收到连接池中
    // 如果type=“UNPOOLED”，代表直连，close则会调用Connection.close()方法
    sqlSession.close();
  }
}
```

## 初始化工具类MybatisUtils

在/src/main/java/com.pfeiking.mybatis.utils下创建MyBatisUtils类

```java
package com.pfeiking.mybatis.utils;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;
import java.io.Reader;

public class MyBatisUtils {
    // 利用static属于类不属于对象，且全局唯一
    private static SqlSessionFactory sqlSessionFactory = null;

    // static块用于初始化静态对象
    static {
        Reader reader = null;
        try {
            reader = Resources.getResourceAsReader("mybatis-config.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        } catch (IOException e) {
            e.printStackTrace();
            throw new ExceptionInInitializerError(e);
        }
    }

    public static SqlSession openSession(){
        return sqlSessionFactory.openSession();
    }

    public static void closeSession(SqlSession sqlSession){
        if (sqlSession != null){
            sqlSession.close();
        }
    }
}
```

```java
@Test
public void testMyBatisUtils() throws Exception {
  SqlSession sqlSession = null;
  try {
    sqlSession = MyBatisUtils.openSession();
    Connection conn = sqlSession.getConnection();
    System.out.println(conn);
  }catch (Exception e){
    throw e;
  }finally {
    MyBatisUtils.closeSession(sqlSession);
  }
}
```



# 2 数据查询

**MyBatis查询步骤**

1. 创建Entity
2. 创建Mapper
3. 编写\<select>SQL标签
4. 开启驼峰命名映射
5. 新增\<mapper>
6. SqlSession执行select语句

## 2.1 创建entity

```java
package com.pfeiking.mybatis.entity;

public class Goods {
    private Integer goodsId;
    private String title;
    private String subTitle;
    private Float originalCost;
    private Float currentPrice;
    private Float discount;
    private Integer isFreeDelivery;
    private Integer categoryId;

    public Integer getGoodsId() {
        return goodsId;
    }

    public void setGoodsId(Integer goodsId) {
        this.goodsId = goodsId;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getSubTitle() {
        return subTitle;
    }

    public void setSubTitle(String subTitle) {
        this.subTitle = subTitle;
    }

    public Float getOriginalCost() {
        return originalCost;
    }

    public void setOriginalCost(Float originalCost) {
        this.originalCost = originalCost;
    }

    public Float getCurrentPrice() {
        return currentPrice;
    }

    public void setCurrentPrice(Float currentPrice) {
        this.currentPrice = currentPrice;
    }

    public Float getDiscount() {
        return discount;
    }

    public void setDiscount(Float discount) {
        this.discount = discount;
    }

    public Integer getIsFreeDelivery() {
        return isFreeDelivery;
    }

    public void setIsFreeDelivery(Integer isFreeDelivery) {
        this.isFreeDelivery = isFreeDelivery;
    }

    public Integer getCategoryId() {
        return categoryId;
    }

    public void setCategoryId(Integer categoryId) {
        this.categoryId = categoryId;
    }
}
```

## 2.2 创建Mapper和编写\<select>SQL标签

通过Mapper将数据库中的字段和entity进行一一对应。需要在/src/main/resources/下创建mappers/goods.xml进行配置。

```java
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="goods">
    <!-- 创建select标签；id就是下面sql的别名；resultType数据放回的对象 -->
    <select id="selectAll" resultType="com.pfeiking.mybatis.entity.Goods">
        select * from t_goods order by goods_id desc limit 10
    </select>
</mapper>
```

```java
@Test
public void testSelectAll() throws Exception {
  SqlSession session = null;
  try{
    session = MyBatisUtils.openSession();
    List<Goods> list = session.selectList("goods.selectAll");
    for (Goods good: list){
      System.out.println(good.getTitle());
    }
  }catch(Exception e){
    throw e;
  }finally{
    MyBatisUtils.closeSession(session);
  }
}
```

## 2.3 开启驼峰命名映射

为了上entity中的goodId和数据库中的good_id对应上，需要在mybatis-config.xml中添加一个设置项

```xml
<settings>
  <!-- goods_id -> goodsId 驼峰命名转换-->
  <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```

## 2.4 SQL传参数

### 查询 \<select>

### 传递单个参数，需要在goods.xml中添加select语句

```xml
<select id="selectById" parameterType="Integer" resultType="com.pfeiking.mybatis.entity.Goods">
  select * from t_goods where goods_id = #{value}
</select>
```

```java
@Test
public void testSelectById() throws Exception {
  SqlSession session = null;
  try{
    session = MyBatisUtils.openSession();
    Goods goods = session.selectOne("goods.selectById", 1602);
    System.out.println(goods.getTitle());
  }catch(Exception e){
    throw e;
  }finally{
    MyBatisUtils.closeSession(session);
  }
}
```

### 传递多个参数

```xml
<select id="selectByPriceRange" parameterType="java.util.Map" resultType="com.pfeiking.mybatis.entity.Goods">
  select * from t_goods
  where
  current_price between #{min} and #{max}
  order by current_price
  limit 0, #{limt}
</select>
```

```java
@Test
public void testSelectByPriceRange() throws Exception {
  SqlSession session = null;
  try{
    session = MyBatisUtils.openSession();
    Map param = new LinkedHashMap();
    param.put("min", 100);
    param.put("max", 500);
    param.put("limt", 10);
    List<Goods> list = session.selectList("goods.selectByPriceRange", param);
    for (Goods good: list){
      System.out.println(good.getTitle()+" "+good.getCurrentPrice());
    }
  }catch(Exception e){
    throw e;
  }finally{
    MyBatisUtils.closeSession(session);
  }
}
```

### 获取多表关联查询结果

- 利用LinkedHashMap保存多表联合结果
- MyBatis会将每一条记录包装为LinkedHashMap对象
- key是字段名，value是字段名对应的值，字段类型根据表结构进行自动判断
- 优点：易于拓展，易于使用
- 缺点：太灵活，无法进行编译时检查

```xml
<select id="selectGoodsMap" resultType="java.util.LinkedHashMap">
    select g.*, c.category_name ,'1' as test from t_goods g, t_category c
    where g.category_id = c.category_id
</select>
```

```java
@Test
public void testSelectGoodsmap() throws Exception {
    SqlSession session = null;
    try{
        session = MyBatisUtils.openSession();
        List<Map> list = session.selectList("goods.selectGoodsMap");
        for (Map map: list){
            System.out.println(map);
        }
    }catch(Exception e){
        throw e;
    }finally{
        MyBatisUtils.closeSession(session);
    }
}
```

## 2.5 ResultMap结果映射

- 可以将查询结果映射为复杂类型的Java对象
- 适用于Java对象保存多表关联结果
- 支持对象关联查询等高级特性

创建com.pfeiking.mybatis.dto，DTO数据传输对象，对原始对象进行拓展，用于数据保存和拓展。

```java
package com.pfeiking.mybatis.dto;

import com.pfeiking.mybatis.entity.Goods;

public class GoodsDTO {
    private Goods goods = new Goods();
    private String categoryName;
    private String test;

    public Goods getGoods() {
        return goods;
    }

    public void setGoods(Goods goods) {
        this.goods = goods;
    }

    public String getCategoryName() {
        return categoryName;
    }

    public void setCategoryName(String categoryName) {
        this.categoryName = categoryName;
    }

    public String getTest() {
        return test;
    }

    public void setTest(String test) {
        this.test = test;
    }
}
```

```xml
<resultMap id="rmGoods" type="com.pfeiking.mybatis.dto.GoodsDTO">
  <!-- 设置主键字段与属性映射-->
  <id property="goods.goodsId" column="goods_id"></id>
  <!-- 设置非主键字段属性映射-->
  <result property="goods.title" column="title"></result>
  <result property="goods.originalCost" column="origin_cost"></result>
  <result property="goods.currentPrice" column="current_price"></result>
  <result property="goods.discount" column="discount"></result>
  <result property="goods.isFreeDelivery" column="is_free_delivery"></result>
  <result property="goods.categoryId" column="category_id"></result>
  <!-- GoodsDTO中的属性-->
  <result property="categoryName" column="category_name"></result>
  <result property="test" column="test"></result>
</resultMap>
<select id="selectGoodsDTO" resultMap="rmGoods">
  select g.*, c.category_name ,'1' as test from t_goods g, t_category c
  where g.category_id = c.category_id
</select>
```

```java
@Test
public void testSelectGoodsDTO() throws Exception {
  SqlSession session = null;
  try{
    session = MyBatisUtils.openSession();
    List<GoodsDTO> list = session.selectList("goods.selectGoodsDTO");
    for (GoodsDTO g: list){
      System.out.println(g.getGoods().getTitle());
    }
  }catch(Exception e){
    throw e;
  }finally{
    MyBatisUtils.closeSession(session);
  }
}
```

# 3 数据写入

**写操作包含如下三种**

- 插入 - \<insert>
- 更新 - \<update>
- 删除 - \<delete>

## 3.1 数据库事务

## 3.2 新增

```xml
<insert id="insert" parameterType="com.pfeiking.mybatis.entity.Goods">
  insert into t_goods(title, sub_title, original_cost, current_price, discount, is_free_delivery, category_id)
  values (#{title}, #{subTitle}, #{originalCost}, #{currentPrice}, #{discount}, #{isFreeDelivery}, #{categoryId})
</insert>
```

```java
@Test
public void testInsert() throws Exception {
  SqlSession session = null;
  try{
    session = MyBatisUtils.openSession();
    Goods goods = new Goods();
    goods.setTitle("测试商品");
    goods.setSubTitle("测试子标题");
    goods.setOriginalCost(200f);
    goods.setCurrentPrice(100f);
    goods.setDiscount(0.5f);
    goods.setIsFreeDelivery(1);
    goods.setCategoryId(43);
    // 返回值为本次成功插入的记录总数
    int num = session.insert("goods.insert", goods);
    System.out.println(num);
    session.commit();//提交事务数据
  }catch(Exception e){
    throw e;
  }finally{
    MyBatisUtils.closeSession(session);
  }
}
```

## 3.3 selectKey与useGeneratedKey

### 3.2.1 区别

- selectKey获取最新加入的数据的主键值，加在\<insert>内，需要明确编写获取最新主键的SQL语句

- useGeneratedKey属性会自动根据驱动生成对应SQL语句

```xml
<insert id="insert" parameterType="com.pfeiking.mybatis.entity.Goods">
  insert into t_goods(title, sub_title, original_cost, current_price, discount, is_free_delivery, category_id)
  values (#{title}, #{subTitle}, #{originalCost}, #{currentPrice}, #{discount}, #{isFreeDelivery}, #{categoryId})
  <selectKey resultType="Integer" keyProperty="goodsId" order="AFTER">
    select last_insert_id()
  </selectKey>
</insert>

<insert id="insert"
        parameterType="com.pfeiking.mybatis.entity.Goods"
        useGeneratedKeys="true"
        keyProperty="goodsId"
        keyColumn="goods_id">
  insert into t_goods(title, sub_title, original_cost, current_price, discount, is_free_delivery, category_id)
  values (#{title}, #{subTitle}, #{originalCost}, #{currentPrice}, #{discount}, #{isFreeDelivery}, #{categoryId})
</insert>
```

```java
@Test
public void testInsert() throws Exception {
  SqlSession session = null;
  try{
    session = MyBatisUtils.openSession();
    Goods goods = new Goods();
    goods.setTitle("测试商品");
    goods.setSubTitle("测试子标题");
    goods.setOriginalCost(200f);
    goods.setCurrentPrice(100f);
    goods.setDiscount(0.5f);
    goods.setIsFreeDelivery(1);
    goods.setCategoryId(43);
    // 返回值为本次成功插入的记录总数
    int num = session.insert("goods.insert", goods);
    System.out.println(num);
    session.commit();//提交事务数据
    System.out.println(goods.getGoodsId());
  }catch(Exception e){
    throw e;
  }finally{
    MyBatisUtils.closeSession(session);
  }
}
```

### 3.3.2 应用场景

- selectKey使用于缩手的关系型数据库
- useGenerateKeys只支持“自增主键”类型的数据库

## 3.4 更新

```xml
<update id="update" parameterType="com.pfeiking.mybatis.entity.Goods">
  update t_goods
  set
  title = #{title},
  sub_title = #{subTitle},
  original_cost = #{originalCost},
  current_price = #{currentPrice},
  discount = #{discount},
  is_free_delivery = #{isFreeDelivery},
  category_id = #{categoryId}
  where
  goods_id = #{goodsId}
</update>
```

```java
@Test
public void testUpdate() throws Exception {
  SqlSession session = null;
  try{
    session = MyBatisUtils.openSession();
    Goods goods = session.selectOne("goods.selectById", 739);
    goods.setTitle("更新测试商品");
    int num = session.update("goods.update", goods);
    System.out.println(num);
    session.commit();
  }catch(Exception e){
    throw e;
  }finally{
    MyBatisUtils.closeSession(session);
  }
}
```

## 3.5 删除

```xml
<delete id="delete" parameterType="Integer">
  delete from t_goods where goods_id = #{value}
</delete>
```

```java
@Test
public void testDelete() throws Exception {
  SqlSession session = null;
  try{
    session = MyBatisUtils.openSession();
    int num = session.delete("goods.delete", 739);
    System.out.println(num);
    session.commit();
  }catch(Exception e){
    throw e;
  }finally{
    MyBatisUtils.closeSession(session);
  }
}
```

## 3.6 SQL注入

**定义**：攻击者利用SQL漏洞，绕过系统约束，越权获取数据的攻击方式

**MyBatis的两种传值方式**：

- **${}**文本替换，未经任何处理对SQL文本替换
- **#{}**预编译传值，使用预编译传值可以预防SQL注入，将参数作为字符串放入SQL中，而SQL的占位符也为字符串

**${}**

```sql
select * from t_goods
where title = '' or 1 = 1 or title='xxxxxxxx'
```

**#{}**

```sql
select * from t_goods
where title = "''or 1 = 1 or title='xxxxxxxx'"
```

如果要是插入sql子句的时候，要用**${}**。
