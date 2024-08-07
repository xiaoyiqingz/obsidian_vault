## **导读**

有些时候，我们的一些配置信息需要比较频繁的修改，如果这些配置信息是放在项目中的话，那么就需要经常进行打包部署，所以我们就思考是否可以把这个配置文件外置呢？

## **一、application.properties外置**

大部分的配置信息，我们都是配置在application.properties，那么这个文件是否可以外置呐？这个当然是可以的。

首先在 `application.preperties` 定义一个属性：
`demo.name = hello.01`

在Controller进行使用：

```java
@Value("${demo.name}")
private String demoName;
 
@RequestMapping("/test")
public String test(){
    return this.demoName;
}
```

将项目打成jar包，使用java -jar的方式进行启动：

java -jar springboot-out-properties-0.0.1-SNAPSHOT.jar

此时读取的值是：hello.01。

将项目中的application.properties拷贝出来，放到和jar包同路径下，修改属性值为：
```java
demo.name = hello.02
```

然后使用上面的命令重新启动，看下效果读取的值就是hello.02了，惊不惊喜意不意外，Spring Boot太牛了，jar包同路径下就直接读取了。

如果我们在jar下新建一个config，然后把application.properties放进去的话，使用上面的命令可以识别吗 ？答案是可以的，

SpringApplication 将从 application.properties 以下位置的文件中加载属性并且将其添加到 Spring 的环境当中：

当前目录下的 /config 子目录

classpath根目录

classpath中的 /config 目录

当前目录

如果自定义的目录，比如conf的话，这个时候就不能识别了，但可以使用--spring.config.location进行指定路径，执行命令如下：

java -jar springboot-out-properties-0.0.1-SNAPSHOT.jar--spring.config.location=conf/application.properties

当然也可以使用绝对路径进行指定：

java -jar springboot-out-properties-0.0.1-SNAPSHOT.jar--spring.config.location=/Users/linxiangxian/Downloads/conf/application.properties

## **二、@PropertySource外置**

在项目中，有些配置会自定义propreties文件进行使用，比如定义了demo.properties：

```
demo.nickname = hello.10
demo.weixin = springboot
```

使用@PropertySource指定配置文件：

```java
/**
 * @PropertySource的例子 
 */

@Configuration
@ConfigurationProperties(prefix = "demo")
@PropertySource(value = {"classpath:demo.properties"})
public class DemoProperties {
    private String nickname;    private String weixin; 
    
    public String getNickname() {
            return nickname;
    } 
    
    public void setNickname(String nickname) {
            this.nickname = nickname;    
    } 
    
    public String getWeixin() {
            return weixin;
    } 
    
    public void setWeixin(String weixin) {
            this.weixin = weixin;
	} 
    
    @Override    
    public String toString() {
            return  "DemoProperties{" +
                            "nickname='" 
                            + nickname + '\'' +
                            + ", weixin='" + weixin + '\'' +
                            + '}';    
	}
}
```

那么此时是可以访问到这个配置文件的，打成jar包，执行命令：

java -jar springboot-out-properties-0.0.1-SNAPSHOT.jar

此时返回的值是：hello.10

将demo.properties放到和jar包同路径下，修改demo.name的值为hello.11，执行上面的命令，芭比Q了，结果还是hello.10，说明Spring Boot对于自定义的properties文件并不能自己从外部去寻找。

那对于这个问题咱么破呢？

很简单，@PropertySource支持多配置多个路径，可以这么配置：

```
PropertySource(value = {"classpath:demo.properties","file:./demo.properties"},ignoreResourceNotFound = true)
```

当我们配置多路径，且多路径下配置文件都存在时，SpringBoot会都加载且会覆盖相同内容。所以当我们配置信息只区分外部和内部路径、内容完全相同时，将file路径写在后面就可以了。当我们本地启动时，因为不存在file路径，所以会加载classpath；当jar启动时，file路径会覆盖classpath路径下的内容；

ignoreResourceNotFound = true 一定要加上，否则找不到会报错。加上之后会忽略找不到的配置文件。

此时将配置文件demo.properties放到和jar包同级下就可以了。

## **总结**

### **对于 **application.properties** 文件：**

（1）默认是读取classpath下的application.properties文件。

（2）jar包同级下的application.properties可以直接读取，启动命名不需要做调整。

（3）jar包同级下的config/application.properties，可以直接读取，启动命令不需要调整。

（4）jar包同级下的conf/application.properties，不可以直接读取，需要通过参数--spring.config.location进行指定。

### **对于自定义的 **.properties** 文件：**

（1）默认是读取classpath下的xxx.properties文件。

（2）jar包同级下的xxx.properties不可以直接读取，需要修改代码的配置@PropertySource指定多个路径，期望最终被使用的路径放到最后，因为会覆盖之前读取的配置信息。

Spring Boot将从 application.properties 以下位置的文件中加载属性并且将其添加到 Spring 的环境当中：

当前目录下的 /config 子目录

classpath根目录

classpath中的 /config 目录

当前目录