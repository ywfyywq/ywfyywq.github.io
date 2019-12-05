### Spring Cloud Config Server

#### SpringBoot 版本

```properties
2.1.4.RELEASE
```

#### 添加config-server依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
</dependencies>
```

#### application.properties

```properties
### 端口
server.port = 8888

### 应用名称
spring.application.name = demo-spring-config-server

### 读取本地配置（读取classpath的配置文件）
#spring.profiles.active = native

### 读取本地GIT目录，指定目录需要有git信息
spring.cloud.config.server.git.uri=D:/cloudconfig
```

#### 新建文件夹

```properties
在D盘新建cloudconfig目录
git bash命令行输入初始化命令
git init
```

#### 添加配置文件

```properties
在cloudconfig目录添加以下文件：

application-dev.properties
application-test.properties
application-test.properties
```

#### 添加注解

```java
// 启动类添加以下注解，开启config server
@EnableConfigServer
@SpringBootApplication
```



### Config Client

#### 添加config client依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
</dependencies>
```

#### 添加resource/bootstrap.properties

```properties
### Config Server服务器访问地址
spring.cloud.config.uri = http://localhost:8888
### 配置名称
spring.cloud.config.name = application
### 配置profile
spring.cloud.config.profile = dev
### git分支
spring.cloud.config.label = master
```

