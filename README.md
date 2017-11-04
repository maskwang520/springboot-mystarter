&emsp;&emsp;在我们日常用springboot的开发过程中，经常会遇到使用如下的一个类来代表程序的入口类。即
```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```
&emsp;&emsp;包括我自己，我平常在开发的过程中，并没有去重点关注spring boot的运行原理，大家都是约定俗成的这么去使用。接下的过程中，将会结合源码简单的分析下springboot运行原理。
#### 一.Springboot 自动配置原理分析
&emsp;&emsp; `@SpringBootApplication`注解`@SpringBootApplication`是一个复合注解，它包括`@SpringBootConfiguration`，`@EnableAutoConfiguration`，`@ComponentScan`三个注解。其中最关键的莫过`@EnableAutoConfiguration`这个注解。在它的源码中加入了这样一个注解`@Import({EnableAutoConfigurationImportSelector.class})`，`EnableAutoConfigurationImportSelector`,它使用`SpringFactoriesLoader. loadFactoryNames`方法来扫描`META-INF/spring.factories`文件，此文件中声明了有哪些自动配置。源码如下（我挑选出重要的一部分）
```java
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		try {
			Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			List<String> result = new ArrayList<String>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
				String factoryClassNames = properties.getProperty(factoryClassName);
				result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
			}
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
					"] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```
&emsp;&emsp;我随便查看spring-boot-autoconfigure-1.5.3.RELEASE.jar中的spring.factories,有如下的自动配置。
```ymal
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
```
>上述spring.factories对应key为org.springframework.boot.autoconfigure.EnableAutoConfiguration的值即为启动时候需要自动配置的类。
####二. 实现自定义的starter
1. 首先定义一个基本的对象类，用来接收application.properties里面特定字段的值。
```java
@ConfigurationProperties(prefix = "hello")
public class HelloServiceProperties {
    private static final String MSG="world";
    private String msg=MSG;

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```
* `@ConfigurationProperties(prefix = "hello")`是类型安全的属性获取。在application.properties 中通过hello.msg来设置，如果不设置默认就是“word"。
2. 定义条件类（根据此类的存在与否来创建这个类的Bean，这个类可以是第三方类库的类）。
```java
public class HelloService {
    private String msg;
    public String sayHello(){
        return msg;
    }
    public void setMsg(String msg){
        this.msg=msg;
    }
}
```
3. 自动配置类
```java
@Configuration  //1
@EnableConfigurationProperties(HelloServiceProperties.class)//2
@ConditionalOnClass(HelloService.class)   //3
@ConditionalOnProperty(prefix = "hello",value = "enabled",matchIfMissing = true)  //4
public class HelloServiceAutoConfiguration {

    @Autowired
    private HelloServiceProperties helloServiceProperties;

    @Bean
    @ConditionalOnMissingBean(HelloService.class)  //5
    public HelloService helloService(){
        HelloService helloService=new HelloService();
        helloService.setMsg(helloServiceProperties.getMsg());
        return helloService;
    }
}
```
* ` @Configuration` 它告知 Spring 容器这个类是一个拥有 bean 定义和依赖项的配置类。
* `@EnableConfigurationProperties`的bean可以以标准方式被注册(例如使用 @Bean 方法),即我定义HelloServiceProperties可以作为标准的Bean被容器管理。
* `@ConditionalOnClass`表示该类在类路径下存在，自动配置该类下的Bean。
* `@ConditionalOnProperty`当指定的属性等于指定的值的情况下加载当前配置类，在这里如果`matchIfMissing`如果为false，则在application.properties中必须存在hello.enable(且不能为false)
* ` @ConditionalOnMissingBean()`表示指定的bean不在容器中，则重新新建@Bean注解的类，并交给容器管理。

&emsp;&emsp;配置好之后，我们还需要在`src\main\resources`下新建文件夹`WEB-INF`，再新建文件`spring.factories`里面的内容如下
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.springboot.mystartertool.HelloServiceAutoConfiguration
```
里面指定的类是上面自定义的那个配置类`HelloServiceAutoConfiguration`
>&emsp;&emsp;定义`spring.factories`的原因是因为`@EnableAutoConfiguration`会扫描jar包下所有`spring.factories`文件，从而构造自动配置类。我们使用的时候使用` @Autowired`注入就行。

在以上工作完成后，我们执行如下命令
```shell
    mvn clean install
```
&emsp;&emsp;就将项目打包到本地maven仓库中，有条件的可以安装的到私服中。
####三. 应用自定义starter
1. 首先引入自定义的starter的jar包
```xml
 <!--引入我的start-->
        <dependency>
            <groupId>com.maskwang</groupId>
            <artifactId>Springboot-mystart</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
```
2. 当我们在`application.properties`配置如下
```Groovy
hello.msg=maskwang
```
&emsp;&emsp;我们就可以使用自定义的starter啦。
```java
@RestController
public class HelloController {
    @Autowired
    HelloService helloService;
    @RequestMapping("/hello")
    public String hello() {
        return helloService.sayHello();
    }
}
```
&emsp;&emsp;由于我们没有自定义`HelloService`，所以会配置类会发挥作用，新建一个`HelloService`,并把里面的msg设置成”maskwang"。没有配置msg，则会采用默认的。结果如下

![image.png](http://upload-images.jianshu.io/upload_images/5281821-3db9b52e9cfc819b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

参考文献：
1. [使用 Java 配置进行 Spring bean 管理](https://www.ibm.com/developerworks/cn/webservices/ws-springjava/index.html)
2. Spring boot实战（汪云飞著）
[github地址](https://github.com/maskwang520/springboot-mystarter)