---
layout: post
title: Mybatis缓存学习
categories: [jvm]
description: 关于Mybatis一级、二级缓存自己学习的一些总结
keywords: mybatis
typora-root-url: ../../
---

这是本人在学习Mybatis源码中的一些总结。

### 准备环境

#### mybatis-config.xml配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <setting name="cacheEnabled" value="true"/> <!-- 开启二级缓存 -->
    </settings>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/test"/>
                <property name="username" value="xxxx"/>
                <property name="password" value="xxx"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="DemoMapper.xml"/>
    </mappers>
</configuration>
```
#### DemoMapper.xml配置文件

```xml
<select id="getUserList" resultMap="BaseResultMap" parameterType="java.util.List">
        SELECT *  FROM user
        WHERE user_no IN
        <foreach collection="userNos" item="userNo" open="(" separator="," close=")">
            #{userNo}
        </foreach>
    </select>
```

### Debug开始

```java
{
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        /**
         * 解析 mybatis-config.xml 里面的元素，设置到：Configuration
         */
        SqlSessionFactory sqlSessionFactory =
                new SqlSessionFactoryBuilder().build(inputStream);
        /**
         * 拿到 DefaultSqlSession，注意不是数据库Connection
         */
        SqlSession sqlSession = sqlSessionFactory.openSession();

        /**
         * JDK动态代理创建Mapper对象：org.apache.ibatis.binding.MapperProxy
         * 也就是说所有的调用都会进入：MapperProxy.invoke方法
         *
         * 大概是：根据当前方法的全路径(org.example.UserMapper.selectUser)去查找sql语句
         * 如果开启了<cacheEnabled>,会先查找二级缓存，再查找一级缓存，再查询数据库
         * 如何禁用：
         * <select id="xx" useCache="false" flushCache="true">
         *     useCache=false, 不用二级缓存,但是会默认走session级缓存
         *     flushCache=true, 清除二级和session缓存
         */
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        User user = mapper.getUserByNo(Collections.singletonList("1000500539493"));
        System.out.println(user);
        User a2 = mapper.getUserByNo(Collections.singletonList("1000500539493"));
        System.out.println(a2);

        //当前sqlSession关闭了才会commit(设置)二级缓存
        sqlSession.close();

//        SqlSession sqlSession2 = sqlSessionFactory.openSession();
//        UserMapper mapper2 = sqlSession2.getMapper(UserMapper.class);
//        User user = mapper2.getUserByNo(Collections.singletonList("1000500539493"));
//        System.out.println(a3);

    }
```

#### DefaultSqlSession

```java
public class DefaultSqlSession implements SqlSession {

  private Configuration configuration;
  
  //如果配置文件里面：<setting name="cacheEnabled" value="false"/>
  //false时--> executor = new SimpleExecutor(this, transaction);
  //true时 --> if (cacheEnabled) {
  //    executor = new CachingExecutor(executor); //包装了一下
  //  }
  private Executor executor; 

  private boolean autoCommit;
  private boolean dirty;
```

