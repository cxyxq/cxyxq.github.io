---
layout: post
title: springboot整合常见三方框架
categories: [springboot]
description: springboot整合dubbo,mybatis等,如何打包运行等
keywords: springboot
typora-root-url: ../../
---


SpringBoot可以帮助我们快速开发一个应用服务。下面整理了如何整合常见的三方框架～
## 整合dubbo

### 引入dubbo依赖

```xml
				    <!-- Dubbo Spring Boot Starter -->
            <dependency>
                <groupId>org.apache.dubbo</groupId>
                <artifactId>dubbo-spring-boot-starter</artifactId>
                <version>2.7.5</version>
            </dependency>

            <!-- zk curator的依赖 -->
            <dependency>
                <groupId>org.apache.curator</groupId>
                <artifactId>curator-recipes</artifactId>
                <version>4.2.0</version>
            </dependency>
```

### 配置dubbo属性

```properties
# Spring boot application
spring.application.name=qz-springboot-dubbo-provider
# 扫描dubbo的@Service注解, 或者在启动类上加@EnableDubbo，2者效果一样
dubbo.scan.base-packages=com.qz.springboot.dubbo.provider

# Dubbo Application
dubbo.application.name=${spring.application.name}

# Dubbo Protocol
dubbo.protocol.name=dubbo
dubbo.protocol.port=20880

## 注册中心zookeeper地址
dubbo.registry.address=zookeeper://127.0.0.1:2181
```

### 服务提供方

在接口实现类上添加**dubbo**的注解`@Service`

```java
@Service(version = "1.0.0") //dubbo的注解
public class DemoServiceImpl implements DemoService {
    private Logger logger = LoggerFactory.getLogger(DemoServiceImpl.class);
    /**
     * The default value of ${dubbo.application.name} is ${spring.application.name}
     */
    @Value("${dubbo.application.name}")
    private String serviceName;

    @Override
    public String sayHello(String name) {
        logger.info("sayHello param {}", name);
        return String.format("[%s] : Hello, %s", serviceName, name);
    }
}
```

### 服务消费者

配置dubbo相关属性

```yaml
spring:
  application:
    name: qz-springboot-dubbo-consumer

dubbo:
  registry:
    address: zookeeper://127.0.0.1:2181
  protocol:
    name: dubbo
  consumer:
    check: false
```



用`@Reference`注解来注入接口

```java
@SpringBootApplication
public class Consumer {
    Logger logger = LoggerFactory.getLogger(getClass());
    public static void main(String[] args) {
        SpringApplication.run(Consumer.class, args);
    }

    @Reference(check = false, version = "1.0.0")
    private DemoService demoService;

    @Bean
    public ApplicationRunner runner() {
        return args -> {
            logger.info(demoService.sayHello("cxyxq"));
        };
    }
}
```




## SpringBoot整合Mybatis
### 引入mybatis,mysql依赖
```xml
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>2.1.0</version>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
```
### 配置datasource,mybatis
在`application.yml`中添加datasource、mybatis的配置
```yaml
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/my_dev
    username: root
    password: root
    
mybatis:
  mapper-locations: classpath:/mybatis/mapper/*.xml
```

### 编写mapper接口
创建`UserMapper.java`,示例中采用注解，xml2种方式:
```
public interface UserMapper {
    /**
     * 采用注解方式
     */
    @Select("select * from user where id = #{uid}")
    User getUserById(@Param("uid") Integer id);

    /**
     * 采用xml方式
     */
    List<User> getUserByName(@Param("name") String name);
}
```

### 编写mapper.xml文件
在`resource/mybaits/mapper/`下创建`UserMapper.xml`：
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.xq.springboot.edu.mapper.UserMapper">

    <select id="getUserByName" resultType="com.xq.springboot.edu.api.User">
        select * from user where name like concat('%',#{name,jdbcType=VARCHAR},'%')
    </select>
</mapper>
```

### 新建UserController
在UserController中注入UserMapper：
```java
@RestController
public class UserController {

    @Autowired
    private UserMapper userMapper;

    @RequestMapping("/id/{id}")
    public User getUserById(@PathVariable("id") Integer id) {
        User user = userMapper.getUserById(id);
        return user;
    }

    @RequestMapping("/name/{name}")
    public List<User> getUserByName(@PathVariable String name) {
        List<User> userList = userMapper.getUserByName(name);
        return userList;
    }
}
```

### 配置Mapper接口扫描
在`ApplicationBoot.java`中，添加注解`@MapperScan`:
```java
import org.mybatis.spring.annotation.MapperScan;

@SpringBootApplication
@MapperScan(value = {"com.xq.springboot.edu.mapper"}) //用来生成mapper接口的代理类
public class ApplicationBoot {
```
### 测试mybatis整合
访问地址[http://127.0.0.1:8080/id/1](http://127.0.0.1:8080/id/1),[http://127.0.0.1:8080/name/a](http://127.0.0.1:8080/name/a)进行2个接口的测试。



## 整合freemarker模板引擎

### 引入freemarker依赖
```xml
<!--引入模板引擎freemarker-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-freemarker</artifactId>
        </dependency>
```

### 编写后台controller
新建`FreemarkerController`类,内容如下：
```java
/**
 * 此处应该用 @Controller表示,因为要返回页面
 **/
@Controller
public class FreemarkerController {

    @RequestMapping("/freemarkerDemo")
    public String freemarkerDemo(ModelMap modelMap) {

        modelMap.addAttribute("title", "我从后台来");

        return "hello";
    }

}
```

### 添加`application.yml`
在`resources`目录下创建配置文件`application.yml`.
```yaml
spring:
  freemarker:
    suffix: .ftl
    template-loader-path: "classpath:/templates/"
```

### 添加.ftl文件
freemarker文件默认存放的目录为：`resources/templates`, 文件以`.ftl`结尾。
我们创建`hello.ftl`内容如下：
```html
<html>
<head>
    <title>spring boot freemarker example</title>
</head>
<body>
<h1>Spring Boot Freemarker 示例</h1>
<table>
<#list users as user>
    <tr>
        <td>${user.id}</td>
        <td>${user.name}</td>
    </tr>
</#list>
</table>
</body>
</html>
```

访问[http://127.0.0.1:8080/freemarkerDemo](http://127.0.0.1:8080/freemarkerDemo)后，即可看到页面输出：  
```
Spring Boot Freemarker 示例
我从后台来
```

### 如何访问静态资源
在`resources/`目录下创建目录`static`,新建`hello.js`:
```
function hello() {
    //do something
}
```
如何访问呢？ 
答案是: [http://127.0.0.1:8080/hello.js](http://127.0.0.1:8080/hello.js),可以看到路径并不需要加:`/static/`。

### 整合jsp的注意事项
项目类型得是`war`包类型,否则无法访问到页面.



## 应用监控

添加依赖:

```xml
						<dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-actuator</artifactId>
                <version>2.2.4.RELEASE</version>
            </dependency>
```





## 整合定时任务

### 注解说明(`@Scheduled`,`@EnableScheduling`)
查看Spring文档[注解方式支持定时任务](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#scheduling-annotation-support),所以需要2个注解：`@Scheduled`和`@EnableScheduling`,其中`@EnableScheduling`需要加到`@Configuration`标注的类上面。
### 添加注解`@EnableScheduling`
在`ApplicationBoot`类上加注解`@EnableScheduling`:
```
@EnableScheduling
public class ApplicationBoot {}
```
### 创建定时任务
创建`ScheduleDemo`类,每隔5s打印当前时间：
```
@Component
public class ScheduleDemo {
    private SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    @Scheduled(fixedRate = 5000)
    public void printTime() {
        System.out.println("Time Now: " + sdf.format(new Date()));
    }
}
```
重新运行当前应用，查看控制台日志输出: `Time Now: 2019-10-13 18:07:15`



## SpringBoot整合多数据源

## 多数据源事务

## SpringBoot全局异常
### 返回自定义的Json体
主要是用`ControllerAdvice`、`ExceptionHandler`来返回自定义异常.
```
@ControllerAdvice
public class WebExceptionHandler {
    @ExceptionHandler(Exception.class)
    @ResponseBody
    public Map<String, Object> globalExceptionHandler(Exception ex) {
        Map<String, Object> map = new HashMap<>();
        map.put("ret", 0);
        map.put("message", "oops,出错啦! " + ex.getMessage());
        return map;
    }
}
```
### 404,500页面的自定义
在`resource/templates/error/`目录下创建:`[HTTP_STATUS].ftl`,比如：404.ftl,500.ftl即可。

## 多环境配置文件
### spring.profiles.active方式
1. 配置文件制定profiles.同时创建多个`application-[env].yml`,例如：`application-dev.yml`,`application-test.yml`:
```
spring:
  profiles:
    active: dev
```
### Maven Profiles方式
我们可以根据maven的profile来控制，很方便：
#### resource配置
```xml
<build>
        <finalName>springboot-edu</finalName>
        <resources>
            <resource> <!-- 定义哪个目录下的文件中的变量,将被profiles替换 -->
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
            </resource>
        </resources>
    </build>
```
#### profiles 多环境配置
```xml
<profiles>
        <profile>
            <id>dev</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <mvn.cxyxq.custom.title>title from maven dev</mvn.cxyxq.custom.title>
                <mvn.cxyxq.custom.message>message from maven dev</mvn.cxyxq.custom.message>
            </properties>
        </profile>
        <profile>
            <id>test</id>
            <properties>
                <mvn.cxyxq.custom.title>title from maven test</mvn.cxyxq.custom.title>
                <mvn.cxyxq.custom.message>message from maven 测试环境</mvn.cxyxq.custom.message>
            </properties>
        </profile>
    </profiles>
```

#### application.yml变量配置
```yml
cxyxq:
  custom:
    title: ${mvn.cxyxq.custom.title}
    message: ${mvn.cxyxq.custom.message}
````

#### Maven 打包
执行maven命令时，可以加`-P`选项来控制用哪个maven profile.  
`mvn -Pdev clean package`  
`mvn -Ptest clean package`

#### 参考文档
[https://portofino.manydesigns.com/en/docs/portofino3/tutorials/using-maven-profiles-and-resource-filtering](https://portofino.manydesigns.com/en/docs/portofino3/tutorials/using-maven-profiles-and-resource-filtering)
[https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html](https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html)

## 如何打包运行

### springboot-maven插件打包

1. 进入代码根目录,执行maven命令`mvn clean package`进行打包
2. 执行完后,进入target目录,执行`java -jar xxx-1.0-SNAPSHOT.jar` 
3. 2.1 发现会报错：`xxx-1.0-SNAPSHOT.jar中没有主清单属性`，表示没有main class. 
    2.2 在`pom.xml`中添加如下内容,然后重新执行步骤1.  

    ```xml
    <build>
    	<plugins>
    		<plugin>
    			<groupId>org.springframework.boot</groupId>
    			<artifactId>spring-boot-maven-plugin</artifactId> 
          <configuration>
              <!-- 如果不指定mainClass,插件会自动搜索带有main方法的类 -->
              <mainClass>xxxx.Application</mainClass>
          </configuration>
    		</plugin>
    	</plugins>
    </build>
    ```

4. 重新打包后，再次运行，即可看到正确输出

### Maven assembly 打包

使用assembly自定义打包

```xml
<!-- 添加plugin -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-assembly-plugin</artifactId>
  <version>2.4</version>
  <configuration>
    <appendAssemblyId>false</appendAssemblyId>
    <descriptors>
      <descriptor>src/main/maven/assembly-package.xml</descriptor>
    </descriptors>
  </configuration>
  <executions>
    <execution>
      <phase>package</phase>
      <goals>
        <goal>single</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

自定义描述符文件,将文件放到目录：`src/main/maven/assembly-package.xml`

```xml
<assembly>
    <id>package</id>
    <formats>
        <format>zip</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <fileSets>
        <fileSet>
            <directory>src/main/bin</directory>
            <outputDirectory>bin</outputDirectory>
        </fileSet>
        <fileSet>
            <directory>src/main/resources</directory>
            <outputDirectory>/conf</outputDirectory>
            <filtered>true</filtered>
        </fileSet>
        <fileSet>
            <directory>lib</directory>
            <outputDirectory>lib</outputDirectory>
        </fileSet>
    </fileSets>
    <dependencySets>
        <dependencySet>
            <outputDirectory>lib</outputDirectory>
        </dependencySet>
    </dependencySets>
</assembly>
```

执行：`mvn clean package`，会在target目录zip包

## 自定义Starter

## SpringBoot启动流程简要说明