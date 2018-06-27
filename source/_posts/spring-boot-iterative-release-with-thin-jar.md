---
title: "SpringBoot: 迭代发布下的Jar瘦身实践"
date: 2018-06-26 15:41:12
tags: ["SpringBoot", "Release"]
description: 随着 Spring Boot 的流行，越来越多开发者选择使用Spring Boot来发布Web应用。不同于传统的War包发布，Spring Boot 把整个项目打包成一个可运行的Jar包（即所谓的Flat Jar），导致了这个Jar包很大（通常有40M+）。如今迭代发布时常有的事情，每次都上传一个如此庞大的文件，会浪费很多时间。<br><br>下面就以一个小项目为例，简述小弟所用的瘦身方案。当然如果是内网发布或者你用的宽带异常给力，瘦身就没有多大意义了。
---

# 项目简介
新建一个练习用的项目，其结构如下
![这里写图片描述](https://nerve-images.oss-cn-shenzhen.aliyuncs.com/github-blog/spring-boot-iterative-release/20170107095138803.png)

1. `ht-cdn`存放的是静态资源（如第三方js、css、images等）
2. `ht-domain`项目中的数据实体定义
3. `ht-repository`数据层接口及实现
4. `ht-service`业务逻辑接口及实现
5. `ht-ui-web`Web管理

其中`ht-ui-web`依赖于`ht-domain`、`ht-repository`、`ht-service`，实现了一个简单的`GetMapping`。

接着打包项目，整个jar包24M这样
![这里写图片描述](https://nerve-images.oss-cn-shenzhen.aliyuncs.com/github-blog/spring-boot-iterative-release/20170107103809887.png)

# 瘦身准备
首先我们要对Jar包有一个初步认识，它的内部结构如下

```text
example.jar
 |
 +-META-INF
 |  +-MANIFEST.MF
 +-org
 |  +-springframework
 |     +-boot
 |        +-loader
 |           +-<spring boot loader classes>
 +-BOOT-INF
    +-classes
    |  +-mycompany
    |     +-project
    |        +-YourClasses.class
    +-lib
       +-dependency1.jar
       +-dependency2.jar
```

运行该Jar时默认从`BOOT-INF/classes`加载class，从`BOOT-INF/lib`加载所依赖的Jar包。如果想要加入外部的依赖Jar，可以通过设置环境变量`LOADER_PATH`来实现。

如此一来，就可以确认我们的思路了：
1. 把那些不变的依赖Jar包（比如spring依赖、数据库Driver等，这些在不升级版本的情况下是不会更新的）从Flat Jar中抽离到单独的目录，如libs
2. 在启动Jar时，设置`LOADER_PATH`使用上一步的libs

这样，我们最终打包的jar包体积就大大减少，每次迭代后只需要更新这个精简版的Jar即可。

# 具体步骤
## 打包时瘦身

通常我们是用`spring-boot-maven-plugin`进行打包，通过阅读文档发现可以通过配置使得该插件在打包时忽略特定的依赖，文档在[spring-boot-maven-plugin](https://docs.spring.io/spring-boot/docs/current/maven-plugin/examples/exclude-dependency.html)。

首先备份一下原先的依赖：编译打包成Flat Jar，然后将`BOOT-INF/lib`下的除去`ht-*`相关的jar文件全部解压出来，另存到`libs`目录下。

接着修改`pom.xml`配置如下

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <layout>ZIP</layout>
                <!--
                    这里可以使用 
                    excludeGroupIds 去除在生产环境中不变的依赖
                    includes        保留打包到 jar 的依赖（通常是项目中的子模块）
                -->
                <!-- <excludeGroupIds>
                    org.springframework.boot,
                    org.springframework,
                    org.springframework.data,
                    org.mongodb,
                    com.github.0604hx,
                    com.fasterxml.jackson.core,
                    commons-beanutils,
                    commons-codec,
                    org.apache.commons,
                    org.apache.tomcat.embed,
                    org.hibernate,
                    org.slf4j,
                    com.jayway,
                    org.jboss,
                    com.alibaba,
                    com.fasterxml,
                    commons-collections,
                    ch.qos.logback,
                    org.scala-lang,
                    org.yaml,
                    org.jboss.logging,
                    javax.validation,
                    nz.net.ultraq.thymeleaf,
                    org.thymeleaf,
                    ognl,
                    org.unbescape,
                    org.javassist
                </excludeGroupIds> -->
                <includes>
                    <include>
                        <groupId>com.nerve</groupId>
                        <artifactId>ht-cdn</artifactId>
                    </include>
                    <include>
                        <groupId>com.nerve</groupId>
                        <artifactId>ht-domain</artifactId>
                    </include>
                    <include>
                        <groupId>com.nerve</groupId>
                        <artifactId>ht-repository</artifactId>
                    </include>
                    <include>
                        <groupId>com.nerve</groupId>
                        <artifactId>ht-service</artifactId>
                    </include>
                </includes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

此时打包好的`ht-ui-web.jar`只有117kb这样
![这里写图片描述](https://nerve-images.oss-cn-shenzhen.aliyuncs.com/github-blog/spring-boot-iterative-release/20170107132421135.png)

BOOT-INF/lib下只有`ht`相关的jar
![这里写图片描述](https://nerve-images.oss-cn-shenzhen.aliyuncs.com/github-blog/spring-boot-iterative-release/20170107132617667.png)

## 运行

由于没有其他依赖，`ht-ui-web.jar`是不能如期运行起来的

```
java -jar ht-ui-web-1.0.jar
Exception in thread "main" java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(Unknown Source)
        at java.lang.reflect.Method.invoke(Unknown Source)
        at org.springframework.boot.loader.MainMethodRunner.run(MainMethodRunner.java:48)
        at org.springframework.boot.loader.Launcher.launch(Launcher.java:87)
        at org.springframework.boot.loader.Launcher.launch(Launcher.java:50)
        at org.springframework.boot.loader.PropertiesLauncher.main(PropertiesLauncher.java:521)
Caused by: java.lang.NoClassDefFoundError: org/springframework/boot/SpringApplication
        at com.nerve.huotong.web.WebApplication.main(WebApplication.java:21)
        ... 8 more
Caused by: java.lang.ClassNotFoundException: org.springframework.boot.SpringApplication
        at java.net.URLClassLoader.findClass(Unknown Source)
        at java.lang.ClassLoader.loadClass(Unknown Source)
        at org.springframework.boot.loader.LaunchedURLClassLoader.loadClass(LaunchedURLClassLoader.java:94)
        at java.lang.ClassLoader.loadClass(Unknown Source)
        ... 9 more
```

至此我们要设置`LOADER_PATH`，如下

```
java -Dloader.path=libs -jar ht-ui-web.jar
```
便可以看到熟悉的Spring Boot 启动信息了。

# 继续瘦身

上面的项目结构介绍提到了`ht-cdn`，我是把前端用到的库都放在这里。然后单独启动一个Web Application。其他项目需要用到静态资源就直接使用。

还有另外一个做法是，把`resources/public`直接丢到`libs`下（就是跟上一步剥离出来的jar包放在一起），结构如下：

![这里写图片描述](https://nerve-images.oss-cn-shenzhen.aliyuncs.com/github-blog/spring-boot-iterative-release/20170107154426390.png)

这样也是可以的（不过要注意不能跟真实项目中自己写的静态资源重名）。

# 结语

经过上面的瘦身，每次迭代开发开的Jar包就显得苗条多了。


----------
插些题外话

**Spring Boot 中的 banner.txt**

banner是spring boot 应用启动时打印在控制台的彩蛋信息，默认是这样的

```

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.4.3.RELEASE)
```

想要修改这个文本的话，只需要在`resources`下新建`banner.txt`即可。这里可以自定义banner：[http://patorjk.com/software/taag](http://patorjk.com/software/taag)

