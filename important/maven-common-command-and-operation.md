---
title: Maven常用命令和操作
date: 2016-08-19 10:15:15
tags: 
- maven
- tools
categories: maven
---
#### Maven Archetypes
Maven archetypes是一个项目模板，可以让Maven按照指定的模板构建出一个项目最基本的文件结构，以及创建一些文件。Maven中有许多项目模板供我们使用，使用方式如下:

```
mvn archetype:generate
```
执行完上述命令之后，Maven会将当前可用的模板全部列出，我这边是一共列出了1648条。
![](/images/maven/maven-command-and-operation-0.png)
你可以输入一个模板前面的数字或者一个过滤器(按照提示)选择你想要的模板，但这样不怎么方便去找到自己想要的模板。这里有3种解决的办法:

1. 你可以将结果导入到一个文本文件中，然后在文件中可以通过关键字查找你所需要的模板信息。例如：
```
mvn archetype:generate > archetypes.txt
```
2.`mvn archetype:create -DgroupId=[your group id] -DartifactId=[your archetype id] -DarchetypeArtifactId=maven-archetype-webapp`指定`archetypeArtifactId`,这样的话就会直接按照所指定的模板构建项目结构。
3.`mvn archetype:generate -DarchetypeCatalog=internal`其中的-DarchetypeCatalog=internal表明使用内置的模板（默认使用中央仓库的模板），这样的话只会列出一些内置的模板，这些内置的模板一般情况下都是能够满足使用的。
![](/images/maven/maven-command-and-operation-1.png)
现在只列出了10条模板，选择第10条模板，然后构建一个webapp的项目结构。
![](/images/maven/maven-command-and-operation-2.png)
按照提示填写一些项目的信息，然后回车，项目结构构建完成，在当前目录下会出现一个以项目名命名的文件夹。
![](/images/maven/maven-command-and-operation-3.png)

#### 编译项目
进入到创建好的项目`cd maven-webapp/`,执行如下命令:`mvn compile`。成功之后，在项目根目录会出现一个`targer`文件夹。因为没有java文件，所以其中的`classes`文件夹为空。
![](/images/maven/maven-command-and-operation-4.png)

#### 打包
执行`mvn package`,`target`文件夹的内容如下。
![](/images/maven/maven-command-and-operation-5.png)
WAR包解压之后就是一个`maven-webapp`文件夹。
`maven-archiver`文件夹中的`pom.properties`放了一些项目的基本信息:
![](/images/maven/maven-command-and-operation-6.png)
假如该项目引入了其他依赖，则会将所有需要的JAR包放置在WEB-INF文件夹下的lib文件夹中。
#### 部署到本地仓库
执行`mvn install`,在本地仓库中出现了该项目的一个库。
![](/images/maven/maven-command-and-operation-7.png)
如果项目产生的是JAR包，则可以在本地的项目中引入并使用该依赖。

#### 部署到github远程仓库
首先在Maven的`setting.xml`中按如下方式配置：
```xml
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>YOUR-GITHUB-USERNAME</username>
      <password>YOUR-GITHUB-PASSWORD</password>
    </server>
  </servers>
</settings>
```
`id`可以随意取，在`pom.xml`中通过该id来引用该信息。
在`pom.xml`中按如下方式配置:

```xml
<properties>
    <!-- github server corresponds to entry in ~/.m2/settings.xml -->
    <github.global.server>github</github.global.server>
</properties>
<distributionManagement>
    <repository>
        <id>internal.repo</id>
        <name>Temporary Staging Repository</name>
        <url>file://${project.build.directory}/mvn-repo</url>
    </repository>
</distributionManagement>
 
<build>
 <plugins>
    <plugin>
        <artifactId>maven-deploy-plugin</artifactId>
        <version>2.8.2</version>
        <configuration>
               <altDeploymentRepository>internal.repo::default::file://${project.build.directory}/mvn-repo</altDeploymentRepository>
        </configuration>
    </plugin>
    <plugin>
         <groupId>com.github.github</groupId>
         <artifactId>site-maven-plugin</artifactId>
         <version>0.12</version>
         <configuration>
              <!-- git commit message -->
              <message>Maven artifacts for ${project.version}</message>
              <!-- disable webpage processing -->
              <noJekyll>true</noJekyll>
              <!-- matches distribution management repository url above -->
              <outputDirectory>${project.build.directory}/mvn-repo</outputDirectory>
              <!-- remote branch name -->
              <!-- <branch>refs/heads/master</branch> -->
              <!-- If you remove this then the old artifact will be removed and new 
               one will replace. But with the merge tag you can just release by changing 
                                                the version -->
              <merge>true</merge>
              <includes>
                <include>**/*</include>
                </includes>
                <!-- github repo name -->
                <repositoryName>项目名</repositoryName>
                <!-- github username -->
                <repositoryOwner>用户名</repositoryOwner>
          </configuration>
          <executions>
              <execution>
                    <goals>
                         <goal>site</goal>
                    </goals>
                    <phase>deploy</phase>
              </execution>
          </executions>
</plugin>
 </plugins>
</build>
```
以上配置中需要修改的地方只有2个地方，github上的项目名称以及你的用户名（注意不是邮箱）。

```xml
				   <!-- github repo name -->
                <repositoryName>项目名</repositoryName>
                <!-- github username -->
                <repositoryOwner>用户名</repositoryOwner>
```
然后执行`mvn clean deploy`，在你的`target`目录下会生成一个名为`mvn-repo`的文件夹，这就是生成的库，不出意外的话，在你的指定的github项目上的`gh-pages`分支（默认）会出现一个Maven库。你也可以指定某个分支,比如master:

```
 <!-- remote branch name -->
              <branch>refs/heads/master</branch>
```

#### 引用github远程仓库的依赖
`pom.xml`按如下方式配置：

```xml
<repositories>
    <repository>
        <id>不重样即可</id>
        <url>https://raw.github.com/用户名/项目名/</url>
        <snapshots>
            <enabled>true</enabled>
            <updatePolicy>always</updatePolicy>
        </snapshots>
    </repository>
</repositories>

<dependencies>
    <dependency>
      <groupId>xxx</groupId>
      <artifactId>xxx</artifactId>
      <version>xxx</version>
    </dependency>
  </dependencies>
```
这是我在`github`上的远程仓库，一个小demo。https://github.com/vinoZHU/my-maven-repo

#### 常用命令整理
以下常用命令摘自:http://lychie.github.io/pages/articles/maven/15040920.html

- test  [ 运行测试 ]
`mvn test`
- -e  [ 显示错误详细信息 ]
`mvn -e test`
- site  [ 生成站点文件 ]
`mvn site` ( 见 target/site 目录 )
- package  [ 打包 ]
`mvn package`
- install  [ 安装到本地仓库 ]
`mvn install` ( 见本地仓库 ${groupId}/${artifactId} 目录 )
- compile  [ 编译源代码 ]
`mvn compile`
- test-compile  [ 编译测试代码 ]
`mvn test-compile`
- clean  [ 清除目标目录产生的结果 ]
`mvn clean`
- -Dmaven.test.skip  [ 跳过测试 ]
`mvn -Dmaven.test.skip package`
- -Dmaven.test.failure.ignore  [ 忽略测试失败 ]
`mvn -Dmaven.test.failure.ignore package`
- dependency:sources  [ 依赖包源码 ]
`mvn dependency:sources`
- dependency:tree  [ 项目依赖树 ]
`mvn dependency:tree`
- project-info-reports:dependencies  [ 项目依赖报告 ]
`mvn project-info-reports:dependencies`( 见 target/site/dependencies.html )
- help:effective-pom  [ 查看有效的 pom 配置 ]
`mvn help:effective-pom` ( 暴露 super pom )
- help:describe  [ 获取插件帮助 ]
`mvn help:describe -Dplugin=compiler -Dmojo=compile -Dfull`
- dependency:resolve  [ 已解决的依赖列表 ]
`mvn dependency:resolve`
- idea:idea  [ 转换成 idea 项目 ]
`mvn idea:idea`
- eclipse:eclipse  [ 转换成 eclipse 项目 ]
`mvn eclipse:eclipse`
- archetype:create  [ 创建 java 项目 ]
`mvn archetype:create -DgroupId=org.lychie -DartifactId=myapp`
- archetype:create  [ 创建 web 项目 ]
`mvn archetype:create -DgroupId=org.lychie -DartifactId=myproject -DarchetypeArtifactId=maven-archetype-webapp`

#### 总结
本文主要整理了Maven的一些常见操作和命令，未整理的部分日后再补充吧。

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流。转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 






