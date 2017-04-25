# Spring Boot 集成 RabbitMQ 实战

## Mac 上 RabbitMQ 的安装
![使用 brew 命令安装 RabbitMQ](http://images2015.cnblogs.com/blog/592770/201704/592770-20170425230533615-2066076509.png)

这样子安装的话，RabbitMQ 的脚本是安装到 /usr/local/sbin 这个目录里的，并且不会自动添加到你的 PATH 里，所以要先添加下。
```
PATH=$PATH:/usr/local/sbin
export PATH=/usr/local/bin:/usr/local/sbin:${PATH}
```

补充说明：sublime .zshrc ，.zshrc 这个文件可以配置环境变量。

## 启动命令
```
rabbitmq-server
```
我们将会看到：
![](http://images2015.cnblogs.com/blog/592770/201704/592770-20170425231915850-848352340.png)

在浏览器中输入 http://localhost:15672/，用户名和密码都是 guest 。

## Spring Boot 集成 RabbitMQ 实战

下面我们使用 Spring Boot 集成 RabbitMQ 模块，初步体验一下 MQ。以下的例子只是一个 HelloWorld ，让我们简单认识 RabbitMQ 的发送消息和接收消息。

示例代码：

特别注意：Gradle 构建中一定要使用 spring-boot 的 plugin。

### 添加 Spring-Boot 插件的方法
在 build.gradle 文件中添加如下配置片段：
```groovy
buildscript {
    ext {
        springBootVersion = '1.4.2.RELEASE'
    }
    repositories {
        // NOTE: You should declare only repositories that you need here
        mavenLocal()
        mavenCentral()
        maven{ url 'http://maven.aliyun.com/nexus/content/groups/public/'}
        maven { url "http://repo.spring.io/release" }
        maven { url "http://repo.spring.io/milestone" }
        maven { url "http://repo.spring.io/snapshot" }
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

apply plugin: 'idea'
apply plugin: 'spring-boot'

```

补充说明：如果想设置阿里巴巴的 nexus 仓库为中央仓库，可以把如下的代码片段粘贴到 build.gradle 文件的第 1 行：
```groovy
allprojects {
    repositories {
        maven{ url 'http://maven.aliyun.com/nexus/content/groups/public/'}
    }
}
```

### 1、引入 spring-boot-starter-amqp 模块；

```groovy
dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.12'
    compile group: 'org.springframework.boot', name: 'spring-boot-starter-amqp', version: '1.3.0.M2'
}
```

还可以配置 application.properties
```text
spring.application.name=rabbitmq-hello
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

### 2、编写 RabbitMQ 的配置类
```java
/**
 * 特别要说明的是：如果在配置文件中声明了 Queue 对象，就不用在 RabbitMQ 的管理员页面创建队列（Queue）了。
 * Created by liwei on 17/3/12.
 */
@Configuration
public class MqConfig {
    /**
     * 声明接收字符串的队列
     *
     * @return
     */
    @Bean
    public Queue stringQueue() {
        return new Queue("my-queue");
    }

    /**
     * 声明接收 User 对象的队列
     *
     * @return
     */
    @Bean
    public Queue userQueue() {
        return new Queue("my-user");
    }
}
```

补充说明：RabbitMQ 管理平台添加队列的方法。
![RabbitMQ 管理平台添加队列的方法](http://images2015.cnblogs.com/blog/592770/201704/592770-20170426002119256-42060858.png)


特别要说明的是：如果在配置文件中声明了 Queue 对象，就不用在 RabbitMQ 的管理员页面创建队列（Queue）了。

### 3、配置 RabbitMQ 的监听器（即接收 MQ 消息的配置）

很简单，使用 `@RabbitListener(queues = "my-queue")` 注解就可以了。

```java
@Component
public class Receiver {

    @RabbitListener(queues = "my-queue")
    public void receiveMessage(String message){
        System.out.println("接收到的字符串消息是 => " + message);
    }


    @RabbitListener(queues = "my-user")
    public void receiveObject(User user){
        System.out.println("------ 接收实体对象 ------");
        System.out.println("接收到的实体对象是 => " + user);
    }
}
```

### 4、编写实体类
特别要注意：实体类一定要实现序列化接口，才可以在网络中传输。
```java
/**
 * 特别注意：实体类一定要实现序列化接口，才可以在网络中传输
 * Created by liwei on 17/4/25.
 */
public class User implements Serializable {

    private final static Long serialVersionUID = 1L;

    private Integer id;
    private String username;
    private String password;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public User() {
    }

    public User(Integer id, String username, String password) {
        this.id = id;
        this.username = username;
        this.password = password;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", username='" + username + '\'' +
                ", password='" + password + '\'' +
                '}';
    }
}
```

### 5、编写启动类

```java
/**
 * 用于测试的控制类
 * Created by liwei on 17/3/12.
 */
@RestController
public class SendMQController {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * http://localhost:8080/send?message=hello
     *
     * @param message
     * @return
     */
    @RequestMapping("/send")
    public String sendMQ(String message) {
        rabbitTemplate.convertAndSend("my-queue", message);
        return "ok";
    }

    /**
     * http://localhost:8080/send/user
     *
     * @return
     */
    @RequestMapping("/send/user")
    public String sendUser() {
        User user = new User();
        user.setId(1);
        user.setUsername("liwei");
        user.setPassword("123456");
        rabbitTemplate.convertAndSend("my-user", user);
        return "OK";
    }

}
```

### 6、测试方法
启动 MyRabbitMqApplication 的 Main 方法，容器启动以后，访问：
http://localhost:8080/send?message=hello
http://localhost:8080/send/user
观察控制台的输出。

本文博客园地址：http://www.cnblogs.com/liweipower2015/p/6765809.html