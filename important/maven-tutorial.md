---
title: Maven基础教程
date: 2016-08-18 16:47:15
tags: 
- maven
- tools
categories: maven
---
#### 什么是Maven
Maven最令人印象深刻的也许是它所提供依赖管理，但是Maven的功能远不止这些，Maven是一个java项目构建工具(build tool)。通常来说，一个项目构建工具需要具备以下这些功能甚至更多:

- 在条件允许的情况下能自动生成源代码。
- 根据源代码能自动生成文档
- 能编译源代码
- 对于java项目，要能将编译好的代码打包成JAR包，WAR包等。
- 能将打包后的文件部署在服务器、仓库或者其他地方。
- 。。。

这些步骤虽然也可以人工手动完成，但是效率比不上构建工具，况且人工出错的概率更大。

#### Maven总览--核心概念
##### pom.xml
Maven最核心的就是一个叫做`pom.xml`的文件，使用Maven构建项目时所需要的所有信息都包含在该文件中，例如项目的版本号，需要的依赖包等等。当你执行一条`mvn *`命令时，Maven需要根据`pom.xml`中的描述来进行操作。
##### Build Life Cycles, Phases and Goals
Maven的构建过程被分成几个生命周期，一个生命周期又被分为若干个阶段，同时一个阶段中又会有若干个目标(相当于一个特定任务)。一条Maven命令通常为`mvn *`,其中`*`的内容为某个生命周期的名字，或者某个阶段的名字，又或者某个目标的名字。这三者在后文会给出具体解释。
##### Dependencies and Repositories
所谓依赖就是项目中需要用到的一些第三方库(JAR files)，如果在本地仓库找不到需要的依赖，Maven会从中央仓库下载到本地仓库。具体内容在后文会涉及到。
##### Build Plugins
插件的作用是往构建时的阶段（Phases）中增添一些目标(Goals),相当于做一些额外的任务。Maven提供了一些标准插件，同时你也可以实现自己的插件。更多插件相关信息可查看官网，http://maven.apache.org/plugin-developers/index.html
##### Build Profiles
在开发过程中项目可能需要处于不同的环境，这时候`pom.xml`中的一些配置可能需要修改。`Profiles`的作用就是当你在`pom.xml`中声明了一个`<profile></profile>`标签对之后，标签对里面的内容可以用来替换原本的配置值，具体内容在后文会涉及到。

#### Maven和Ant的区别
Ant也是Apache的一个构建工具，两者之间的最大区别是:Ant必须要求指定具体动作，细化程度达到如拷贝一个文件之类的操作。而Maven只要告诉它要做什么，具体做法在Maven中已经预先在`Phases`和`Goals`中定义过了。Ant强调怎么去做，而Maven则强调做什么。

#### Maven pom.xml分析
pom是（Project Object Model）的缩写，每个Maven项目都有一个`pom.xml`与之对应，文件处于项目的根目录。当一个项目含有多个子项目时，主项目和每个子项目都对应一个`pom.xml`。此时既可以把整个项目一同构建，也可以只根据子项目的`pom.xml`来单独构建每个子项目。

文件内容描述了该Maven项目的本身信息，需要的依赖（JAR files），应该构建哪些内容，需要的资源等等。以下是一个最简化的`pom.xml`文件：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.vino</groupId>
    <artifactId>maven-app</artifactId>
    <version>1.0.0</version>
</project>
```
- `modelVersion`元素指定你所使用的POM模型，4.0.0版本对应Maven 2.x或者3.x版本。
- `groupId `元素对于开源项目(e.g.中央仓库的依赖）来说必须是唯一的，因为Maven寻找依赖时必须根据groupId。对于大多数普通使用者来说，groupId和项目包名类似。如果是个网站项目，也可以是网站的域名。这里提一点，假如想让自己的项目变成一个Maven库，以上`pom.xml`的groupId为`com.vino`,则在Maven库中的文件目录存在形式为:`MAVEN_REPO/com/vino/`。
- `artifactId `元素包含着项目名。将项目打包时，包名就是该元素内的值。该值中的点号不会变成文件分隔符。假如我有2个项目，groupId都为`com.vino`, artifactId一个为`app1`，一个为`maven.app2`,当我将2个项目部署成本地Maven库时，目录形式为:`MAVEN_REPO/com/vino/app1/`、`MAVEN_REPO/com/vino/maven.app2/`。
- `version `元素表明当前项目的版本号，相当于在一个项目下面再划分了一个层次。

使用上述`pom.xml`部署成本地Maven库的文件为:`MAVEN_REPO/com/vino/maven-app/1.0.0/maven-app-1.0.0.jar`

#### Maven依赖
只要不是太小的项目，基本上都需要第三方的JAR包。但是这些JAR包一般都是有不同的版本的，如果手动更新、管理这些JAR包，必定又麻烦又费时。Maven内置了依赖管理，只需要在`pom.xml`中声明，在构建项目时Maven便会将依赖从中央仓库下载到你的本地仓库。如果这些第三方的依赖同时也需要一些其他依赖，Maven会将这些依赖统统下载到你的本地仓库。以下是使用的一个样例:
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
   http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.vino</groupId>
    <artifactId>maven-app</artifactId>
    <version>1.0.0</version>
    
      <dependencies>

        <dependency>
          <groupId>org.jsoup</groupId>
          <artifactId>jsoup</artifactId>
          <version>1.7.1</version>
        </dependency>

        <dependency>
          <groupId>junit</groupId>
          <artifactId>junit</artifactId>
          <version>4.8.1</version>
          <scope>test</scope>
        </dependency>

      </dependencies>
    

    <build>
    </build>

</project>
```
项目中需要的每个依赖，都放在一个`<dependency></dependency>`标签对中，并且用`groupId`,`artifactId`,`version`进行描述。下载到本地之后，两个依赖的目录形式为:`MAVEN_REPO/junit/junit/4.8.1/`、`MAVEN_REPO/org/jsoup/jsoup/1.7.1/`。目录内容中除了JAR包，还有一个该依赖的`pom.xml`文件和一些用于校对内容完整性的文件。

**将本地JAR包作为依赖**
有时候某些开源项目没有放在Maven的中央仓库，需要单独下载。这时候需要将下载到本地的JAR包作为一个项目依赖来使用，具体用法如下:

```xml
<dependency>
  <groupId>mydependency</groupId>
  <artifactId>mydependency</artifactId>
  <scope>system</scope>
  <version>1.0</version>
  <systemPath>${basedir}/war/WEB-INF/lib/mydependency.jar</systemPath>
</dependency>
```
这里需要注意的是: `groupId`和`artifactId`可以乱填（但一般是依赖的名字），`scope`必须填`system`,表明是本系统的文件，`version`同样可以乱填，`systemPath`就是JAR包的位置，`${basedir}`代表`pom.xml`所在的目录。

**快照依赖**
所谓快照依赖，就是一些仍处于开发状态的依赖。当使用快照依赖时，你不需要经常性地去手动更新版本号来获得最新的依赖。你可以设置一个时间间隔，Maven会自动的每隔一段时间将依赖下载的本地，即使中央仓库的依赖并未更新。但前提是，该依赖的版本号中的大版本号与`pom.xml`中的对应:

```xml
<dependency>
    <groupId>com.vino</groupId>
    <artifactId>maven-app</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```
假如中央仓库的版本号变成了2.0-SNAPSHOT,则Maven不会下载该依赖。

#### 中央仓库与远程仓库的区别
中央仓库是Maven官方的一个仓库，而远程仓库放在一个其他的服务器上，Maven可以从该服务器下载依赖。github就可以作为一个远程仓库，当我们把开源项目以约定好的的形式放到到github上之后，可以通过如下方式引用:

```xml
<repositories>
    <repository>
        <id>your-mvn-repository</id>
        <url>https://raw.github.com/yourGitHubId/mvn-repository/master/releases</url>
    </repository>
</repositories>

<dependencies>
</dependencies>
```

#### Maven Build Life Cycles, Phases and Goals
生命周期这个概念听上去感觉很模糊，可以把生命周期理解成一个完整的过程。Maven有3种内置的构建生命周期，相当于3种不同功能的过程:

1. default
2. clean
3. site 

3者之间相互独立，分开执行。


`clean`的功能是就是将已生成的资源文件，编译后文件，JAR包，WAR包等清除，使用方法为:
`mvn clean`。

`site`的功能是将当前项目的信息整理成一个文档，执行`mvn site`后，在项目的`target`目录下会生成一个`site`文件夹，其中的内容便是项目文档的html文件。

`default`生命周期做的事就是Maven主要功能，无法直接执行`mvn default`，只能执行它的阶段(Phases)或者目标(Goals)。

它最常用的阶段(Phases)如下：

|Build Phase|Description|
|:---:|:---|
|validate|验证项目是否无错误，所有的必须的信息都具备了，包括所需要的依赖是否已经下载。|
| compile |编译项目源代码|
| test |运行单元测试中的代码，这些测试的代码不应该没被打包或者部署。|
| package |将编译好的代码打包成某种格式，比如JAR或者WAR等|
|install|将package后的包安装到本地仓库，可以作为一个依赖来使用|
| deploy |将最终的包拷贝到远程仓库，供他人使用|

当我想使用`package`，只需在`pom.xml`目录执行`mvn package`,这时候在项目的`target`下会出现一个WAR包（如果是webapp）。但是我都没编译，怎么就可以打包了呢？事实上，当执行某个阶段命令时，该命令所属的生命周期之前的所有阶段步骤都会按照顺序被执行一遍。所以在`mvn package`被执行时，之前的命令（包括 mvn compile）已经执行过了。

现在构建阶段已经知道了是怎么回事，现在就剩下目标了，目标可以理解为构建阶段中的一个具体任务。比如`dependency:copy-dependencies`就是一个目标（任务），执行`mvn dependency:copy-dependencies`时,会在`target`目录下生成一个`dependency`文件夹，内容为该项目使用的所有依赖(JAR files)。

关于这三者的完整的信息可以查看官网，http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html

#### Maven Build Profiles
上面提到过，profile可以简单地认为是一个环境。使用方法很简单，如下:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
   http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.vino</groupId>
  <artifactId>maven-app</artifactId>
  <version>1.0.0</version>

  <profiles>
      <profile>
          <id>test</id>
          <activation>...</activation>
          <build>...</build>
          <modules>...</modules>
          <repositories>...</repositories>
          <pluginRepositories>...</pluginRepositories>
          <dependencies>...</dependencies>
          <reporting>...</reporting>
          <dependencyManagement>...</dependencyManagement>
          <distributionManagement>...</distributionManagement>
      </profile>
  </profiles>

</project>
```
其中的`activation`用来表明当前profile是否正在使用，如果正在使用，则该profile中的元素会覆盖掉原来的值。

#### 总结

以上是我整理的一些Maven相关的信息，主要是一些基础的原理性的内容。


><font color= Darkorange>如若觉得本文尚可，欢迎转载交流。转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 