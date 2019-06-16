---
title: springboot打包与微服务部署
date: 2019-06-16 10:07:13
tags: "springboot"
category: "微服务"
---

# springboot打包与微服务部署

微服务部署有两种方法：

（1）手动部署：首先基于源码打包生成jar包（或war包）,将jar包（或war包）上传至虚拟机并拷贝至JDK容器。

（2）通过Maven插件自动部署。（常用，和手动部署原理一致）

## 1. 手动部署

### 1. 将源码打包生成jar包

SpringBoot打包

1. pom.xml添加

  ```xml
  <build>
  	<plugins>
  		<plugin>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-maven-plugin</artifactId>
  			<configuration>
  				<!-- Spring Boot的启动类-->
  				<mainClass>com.tensquare.eureka.EurekaApplication</mainClass>
  			</configuration>
  		</plugin>
  	</plugins>
  </build>
  ```

2. 在启动类添加 extends SpringBootServletInitializer 

3. 在启动类中重写配置
  @Override
    protected SpringApplicationBuilder configure(
            SpringApplicationBuilder builder) {
        return builder.sources(this.getClass());
    }

4. clean, package

5. java -jar 打成包的文件名(因为Windows配置了jdk的环境变量，在Windows直接就可以跑起来了，在docker中，需要在jdk镜像基础上打包微服务镜像，下面是在docker中部署微服务的步骤)

### 2.  使用脚本创建jdk镜像 

步骤：

（1）创建目录

```
mkdir –p /usr/local/java
```

（2）下载jdk-8u171-linux-x64.tar.gz并上传到服务器（虚拟机）中的/usr/local/java目录

（3）创建文件Dockerfile  `vi Dockerfile`

```
#依赖镜像名称和ID
FROM centos:7
#指定镜像创建者信息
MAINTAINER ITCAST
#切换工作目录
WORKDIR /usr
RUN mkdir  /usr/local/java
#ADD 是相对路径jar,把java添加到容器中
ADD jdk-8u171-linux-x64.tar.gz /usr/local/java/

#配置java环境变量
ENV JAVA_HOME /usr/local/java/jdk1.8.0_171
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
ENV PATH $JAVA_HOME/bin:$PATH
```

（4）执行命令构建镜像

```
docker build -t='jdk1.8' .
```

（5）查看镜像是否建立完成

```
docker images
```

（6）创建容器

```
docker run -it --name=myjdk8 jdk1.8 /bin/bash
```

### 3. 使用Dockerfile在jdk镜像上创建微服务镜像

```
# 基于哪个镜像
FROM java:8

# 将本地文件夹挂载到当前容器
VOLUME /tmp

# 拷贝文件到容器，也可以直接写成ADD microservice-discovery-eureka-0.0.1-SNAPSHOT.jar /app.jar
ADD microservice-discovery-eureka-0.0.1-SNAPSHOT.jar app.jar
RUN bash -c 'touch /app.jar'

# 开放8761端口
EXPOSE 8761

# 配置容器启动后执行的命令
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

### 4. 将微服务镜像run成容器即可

## 2. 通过Maven插件自动部署

Maven插件自动部署步骤：

### （1）修改宿主机的docker配置，让其可以远程访问

```
vi /lib/systemd/system/docker.service
```

其中ExecStart=后添加配置`-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock`



### （2）刷新配置，重启服务

```shell
#刷新配置
systemctl daemon-reload
#重启docker
systemctl restart docker
#重启私有仓库
docker start registry
```

### （3）在tensquare_eureka工程pom.xml 增加配置

```xml
     <build>
        <finalName>app</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- docker的maven插件，官网：https://github.com/spotify/docker-maven-plugin -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.4.13</version>
                <configuration>
               <imageName>192.168.184.135:5000/${project.artifactId}:${project.version}</imageName>
                    <baseImage>jdk1.8</baseImage>
                    <entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                    <dockerHost>http://192.168.184.135:2375</dockerHost>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

以上配置会自动生成Dockerfile

```
FROM jdk1.8
ADD app.jar /
ENTRYPOINT ["java","-jar","/app.jar"]
```

### （4）在windows的命令提示符下，进入tensquare_eureka工程所在的目录，输入以下命令，进行打包和上传镜像

```
mvn clean package docker:build  -DpushImage

#跳过测试
-DskipTests
```