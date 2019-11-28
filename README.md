# SpringCloudConfig
SpringCloudConfig
---
&emsp;&emsp;**本文简单介绍了SpringCloud的分布式配置中心Config及其简单的使用教程，同时也介绍了基于SpringBoot 1.X、SpringCloud Dalston版本的SpringCloud Config的配置刷新机制Refresh的简单使用，戳[这里](https://github.com/butalways1121/SpringCloudConfig)下载源码。**
<!-- more -->
## 一、SpringCloud Config
### 1.SpringCloud Config介绍
&emsp;&emsp;Spring Cloud Config是一个解决分布式系统的配置管理方案，为分布式系统中的外部配置提供服务器和客户端支持，方便部署与运维。它包含了Client和Server两个部分，服务端也称分布式配置中心，是一个独立的微服务应用，用来连接配置服务器并为客户端提供获取配置信息，加密/解密信息等访问接口；客户端则是通过指定配置中心来管理应用资源以及与业务相关的配置内容，通过接口从配置中心获取和加载配置信息，并在启动的时候依据此数据初始化自己的应用。Spring Cloud Config默认采用Git来存储配置信息，并且可以通过Git客户端工具来方便管理和访问配置内容。
### 2.SpringCloud Config优点
* 集中管理配置文件；
* 不同环境不同配置，动态化的配置更新；
* 运行期间，不需要去服务器修改配置文件，服务会想配置中心拉取自己的信息；
* 配置信息改变时，不需要重启即可更新配置信息到服务；
* 配置信息以 rest 接口暴露。

### 3.为什么要使用SpringCloud Config
首先，先看一下Config的结构，如下图：

![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/87.png)

&emsp;&emsp;随着项目越来越大，模块越来越多，配置文件少则四五个多则二三十个，配置放代码和配置文件里不够安全，放maven里不够优雅，放环境变量里是够安全，随着docker的流行也还算方便，可不能统一管理，况且还要写个脚本往环境变量里写，这都太麻烦了，能不能用一个优雅，安全又方便统一管理的方法来解决这些问题呢，SpringCloud Config就是一个比较好的解决方案。

&emsp;&emsp;从上图中可以看到SpringCloud Config整体结构包括三个部分：客户端（各个微服务应用），服务端（中介者），配置仓库（可以是本地文件系统或者远端仓库，包括git,svn等）。其中，配置仓库中放置各个配置文件（.yml 或者.properties），服务端指定配置文件存放的位置，客户端指定配置文件的名称。

&emsp;&emsp;SpringCloud Config的这种结构配置是集中化管理的，因为是分布式应用，当修改某个应用的配置的时候，就不需要到该应用中去修改相关的配置，并且修改之后还有重启应用，相对来说很麻烦。当迁移仓库的位置时，只需要修改server中的配置即可，client中无需进行任何修改,并且SpringCloud Config还支持热更新，当你修改了配置文件中的配置，通过`post: http://hostname:port/actuator/refresh` 到server应用操作，就可以实现配置热更新，当client中的类使用了`@RefreshScope`注解，那么该类再次使用时，新更改的配置会生效。
## 二、SpringCloud Config 示例
### 1.服务端
&emsp;&emsp;这次的项目需要创建两个服务端springcloud-config-eureka和springcloud-config-server，前者用于注册中心，后者则用于配置中心。
#### （1）springcloud-config-eureka
pom.xml完整配置如下，这里依旧使用的是1.5.9.RELEASE版本的SpringBoot：
```bash
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>1.0.0</groupId>
	<artifactId>springcloud-config-eureka</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>
	<name>springcloud-config-eureka</name>
	<url>http://maven.apache.org</url>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath />
	</parent>
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Dalston.RELEASE</spring-cloud.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```
application.properties文件配置信息：
```bash
spring.application.name=springcloud-config-eureka-server
server.port=8005
eureka.client.register-with-eureka=true
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:8005/eureka/
```
启动类代码示例：
```bash
@SpringBootApplication
@EnableEurekaServer
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class,args);
        System.out.println( "config注册中心服务启动。。。" );
    }
}
```
#### (2)springcloud-config-server
pom.xml文件在springcloud-config-eureka的依赖基础上添加Config服务端的依赖：
```bash
<!-- Config依赖 -->
	<dependency>
	    <groupId>org.springframework.cloud</groupId>
	    <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
```
application.properties文件配置信息：
```bash
spring.application.name=springcloud-config-server
server.port=9005
eureka.client.serviceUrl.defaultZone=http://localhost:8005/eureka/
#配置的Git仓库的地址。
spring.cloud.config.server.git.uri = https://github.com/butalways1121/springcloud-config/
#git仓库地址下的配置文件目录
spring.cloud.config.server.git.search-paths = /config-repo
#git仓库的账号
spring.cloud.config.server.git.username = xxxxx
#git仓库的密码。
spring.cloud.config.server.git.password = xxxxx
```
&emsp;&emsp;**注：以上配置信息是通过Git方式做一个配置中心，然后每个服务从其中获取自身配置所需的参数。SpringCloud Config也支持本地参数配置的获取，如果使用本地存储的方式，在配置文件添加 `spring.profiles.active=native`，取消Git的相关配置，然后在src/main/resources路径下新增一个文件即可，这样定义后就会从项目的resources路径下读取配置文件。另外，如果是读取指定路径下的配置文件，那么可以使用 `spring.cloud.config.server.native.searchLocations = file:D:/properties/ `来读取。**

为进行本地配置文件测试，在resources路径下新建一个configtest.properties配置文件，添加如下内容：
`word=hello world`

启动类代码示例如下，其中`@EnableConfigServer`注解表示启用config配置中心功能：
```bash
@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigServer
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class,args);
        System.out.println( "config配置中心服务端启动成功!" );
    }
}
```
### 2.客户端
&emsp;&emsp;创建一个springcloud-config-client项目用于读取配置中心的配置，pom.xml文件在springcloud-config-eureka的依赖基础上添加Config客户端的依赖，需要注意的是Config服务端和客户端的依赖是不同的：
```bash
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```
新建一个bootstrap.properties文件，并添加如下信息:
```bash
#spring.cloud.config.name： 获取配置文件的名称
spring.cloud.config.name=configtest
#spring.cloud.config.profile: 获取配置的策略,客户端通过spring.cloud.config.profile=pro/dev来指定拉取的环境配置
spring.cloud.config.profile=pro
#spring.cloud.config.label：获取配置文件的分支，默认是master。如果是是本地获取的话，则无用。
spring.cloud.config.label=master
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.serviceId=springcloud-config-server
eureka.client.serviceUrl.defaultZone=http://localhost:8005/eureka/
```
&emsp;&emsp;**注:上面这些与SpringCloud相关的属性必须配置在bootstrap.properties文件中，Config部分的内容才能被正确加载，因为bootstrap.properties的相关配置会先于application.properties，而bootstrap.properties的加载也是先于application.properties。需要注意的是eureka.client.serviceUrl.defaultZone也要配置在bootstrap.properties，不然客户端是无法获取配置中心参数的，会启动失败！**

application.properties配置：
```bash
spring.application.name=springcloud-config-client
server.port=9006
```
启动类代码示例：
```bash
@SpringBootApplication
@EnableDiscoveryClient
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class,args);
        System.out.println( "config配置中心客户端启动成功!" );
    }
}
```
为了方便查询，新建一个ClientController控制类，在其中进行参数的获取并返回：
```bash
@RestController
public class ClientController {

	@Value("${word}")
	private String word;
	
    @RequestMapping("/hello")
    public String index(@RequestParam String name) {
        return name+","+this.word;
    }
}
```
这里，`@Value`注解是默认是从application.properties配置文件获取参数，但是我们在bootstrap.properties文件中使用`spring.cloud.config.name=configtest`指定了获取配置文件的名称，如果是采用本地存储的方式，则直接是从resources路径下或者application.properties文件中指定的路径下的configtest.properties文件中读取配置信息；如果是Git方式，则是从配置中心服务端指定的路径下读取configtest.properties文件中的配置信息。
### 3.测试
#### （1）本地测试
&emsp;&emsp;首先，将springcloud-config-server项目的application.properties配置文件修改为使用本地存储的方式，即注释掉`spring.cloud.config.server.git`相关的配置，添加`spring.profiles.active=native`，然后在src/main/resources下创建configtest.properties文件，然后在里面添加一个配置`word=hello world`。

&emsp;&emsp;修改完成之后，依次启动springcloud-config-eureka、springcloud-config-server、springcloud-config-client这三个项目，接着在浏览器中输入`http://localhost:8005/`查看Eureka注册中心的信息，如下：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/88.png)

然后，获取一下配置中心服务端的配置文件，在浏览器中输入`http://localhost:9005/configtest-1.properties`查看configtest.properties文件的配置信息:
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/89.png)

&emsp;&emsp;**注：从上图访问的地址可以发现，我们定义的配置文件的名称是configtest.properties，访问的配置文件名后面还有一个`-1`（`-`后面可以随意添加一个参数），这其实是Config的访问规则，如果直接使用该名称是获取不到的，因为配置文件名需要通过`-`来进行获取，所以在创建文件的时候，最好是按这种命名规则来创建，如果配置文件名称没有`-`，那么添加了`-`之后，会自动进行匹配搜索。SpringCloud Config 的URL与配置文件的映射关系如下：**
```bash
/{application}/{profile}/[label]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```
&emsp;&emsp;**其中，`{application}`指的是配置文件的前缀，通常我们取微服务的名称好一点；`{lable}`对应Git上不同的分支，默认为master，指定访问某分支下的配置文件；`{profile}`是`-`后面的部分，通常指的是环境信息。上述URL：`http://localhost:9005/configtest-1.properties`会映射{application}-{profile}.properties对应的配置文件。**

然后调用springcloud-config-client客户端的接口，查看是否能够获取配置信息，在浏览器上输入`http://localhost:9006//hello?name=butalways`:
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/90.png)

#### （2）Git测试
&emsp;&emsp;在完成本地测试之后，再将springcloud-config-server项目的application.properties配置文件修改为使用Git方式读取配置文件，即将`spring.profiles.active=native`注释掉，解除`spring.cloud.config.server.git`相关的注释(账号和密码要填写真实的)，然后在Git的springcloud-config仓库下建立一个config-repo文件夹，新建configtest-pro.properties、configtest-dev.properties两个配置文件，内容分别为`word=hello world!!`和`word=hello world!`， 然后和configtest.properties配置文件一起上传到config-repo文件夹中，依次启动三个程序，进行接下来的测试：

首先，在浏览器中输入`http://localhost:9005/configtest-dev.properties`查看configtest-dev.properties文件信息：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/91.png)

再输入`http://localhost:9005/configtest-pro.properties`查看configtest-pro.properties信息：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/92.png)

接着，调用客户端的接口测试，在浏览器中输入`http://localhost:9006/hello?name=butalways`：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/93.png)

由于springcloud-config-server项目的bootstrap.properties文件中配置的是`spring.cloud.config.profile=pro`和`spring.cloud.config.name=configtest`，所以读取的是 configtest-pro.properties文件的信息，想要获取其他的配置，修改以上两项配置即可。
## 三、SpringCloud Refresh
&emsp;&emsp;上文中介绍了SpringCloud Config配置中心的Git使用其实存在一个问题，当我们修改了configtest-pro.properties配置文件并重新提交后，再去调用客户端接口`http://localhost:9006/hello?name=butalways`获取配置信息，会发现得到的是修改之前的信息，只有在客户端重启后才可以获取最新的配置，那么接下来就尝试使用Refresh机制实现客户端主动获取Config Server配置服务中心的最新配置以实现动态更新。

### 1.修改相关配置
&emsp;&emsp;我们直接在上述项目基础上进行修改，首先，要使用Refresh机制需要在客户端项目springcloud-config-client的pom.xml的文件中添加如下配置：
```bash
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
**注：`spring-boot-starter-actuator`表示对程序进行监控，可以通过http接口得到该程序的各种信息。**

接着，到application.properties中添加配置`management.security.enabled=false`，表示暴露所有端点。

最后，在控制层上添加`@RefreshScope`注解即可，该注解表示在接到SpringCloud配置中心配置刷新的时候，自动将新的配置更新到该类对应的字段中。

### 2.测试
&emsp;&emsp;完成修改之后，就来测试SpringCloud Config是否可以进行配置实时更新，首先，依次启动springcloud-config-eureka、springcloud-config-server和springcloud-config-client三个项目，然后在浏览器中输入`http://localhost:9006//hello?name=butalways`查看服务端configtest.properities的配置信息：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/93.png)

接着，把Git端的configtest.properities文件配置改为`word=hello world!!hello world!!`，再请求`http://localhost:9006//hello?name=butalways`，此时浏览器返回的还是修改之前的配置信息，可以发现配置并没有实时刷新，查阅官方文档得知，这里需要客户端通过POST方法触发各自的/Refresh才可以实现动态刷新，所以我们就使用Postman工具模拟post请求地址`http://localhost:9006/refresh`，返回如下说明已经完成了word配置的刷新：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/94.png)

然后，浏览器中再次请求`http://localhost:9006//hello?name=butalways`：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/95.png)

若结果如图，则说明我们已经成功实现了动态刷新配置。
