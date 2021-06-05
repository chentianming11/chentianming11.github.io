---
title: gradle构建spring源码环境
tags: spring
categories: spring
abbrlink: 3719910845
date: 2021-06-05 13:12:44
---

相信每一个`Java`程序员都有阅读`spring`源码的想法，但是在构建的时候就碰到了各种坑，本篇文章详细介绍使用构建`spring-framework-5.0.2.RELEASE-中文注释版`的方式，按照这个步骤保证你一次构建成功，避免踩坑。

<!--more-->

## 下载源码

`spring-framework-5.0.2.RELEASE-中文注释版`链接: [https://pan.baidu.com/s/1lsmuCuZ7znTdjAXOrhm2kg](https://pan.baidu.com/s/1lsmuCuZ7znTdjAXOrhm2kg)，提取码: c7vw 

## 下载对应gradle版本

### 确定gradle版本

不同版本的`spring`，使用的`gradle`版本也不一样。查看`gradle/wrapper/gradle-wrapper.properties`文件可以看到当前`spring`采用的`gradle`是什么。

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https://services.gradle.org/distributions/gradle-4.3.1-bin.zip
```

可以看到，我们使用的`gradle`版本是`4.3.1`。

### 手动安装gradle

gradle下载地址：[https://gradle.org/releases/](https://gradle.org/releases/)

找到`4.3.1`版本进行下载。然后配置环境变量，下面以mac为例说明。

![gradle](https://chentianming11.github.io/images/spring/gradle.png)

首先在命令行终端输入`vim ~/.bash_profile`命令，然后新增以下配置：

```text
export GRADLE_HOME=/Users/chentianming/Downloads/work/gradle-4.3.1
export PATH=$PATH:$GRADLE_HOME/bin
```

最后执行`source ~/.bash_profile`使配置生效。完成之后，使用`gradle -v`验证。

### 配置阿里云镜像源

为了加快访问速度，可全局配置阿里云镜像源。进入`~/.gradle`文件夹，新建`init.gradle`文件，输入以下内容：

```text
allprojects{
    repositories {
        def ALIYUN_REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public'
        def ALIYUN_JCENTER_URL = 'http://maven.aliyun.com/nexus/content/repositories/jcenter'
        all { ArtifactRepository repo ->
            if(repo instanceof MavenArtifactRepository){
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_REPOSITORY_URL."
                    remove repo
                }
                if (url.startsWith('https://jcenter.bintray.com/')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_JCENTER_URL."
                    remove repo
                }
            }
        }
        maven {
            url ALIYUN_REPOSITORY_URL
            url ALIYUN_JCENTER_URL
        }
    }
}
```

## 编译spring

打开`import-into-idea.md`文件，里面描述了构建步骤，重点步骤如下：

1. 使用`gradle :spring-oxm:compileTestJava`预编译。
2. 导入到IDEA中（File -> New -> Project from Existing Sources -> Navigate to directory -> Select build.gradle）。
3. 当提示时排除 spring-aspects 模块。

### 预编译

### 镜像改为阿里云

编译前，将项目的`build.gradle`指定的镜像源改为阿里云，确保依赖可以顺利下载。
![gradle-build](https://chentianming11.github.io/images/spring/build-gradle.png)

### 执行预编译

输入命令执行预编译。如果出现`aspectJ`相关失败，可以把对应的Test代码先注掉。
![](https://chentianming11.github.io/images/spring/spring-test-error1.png)
![](https://chentianming11.github.io/images/spring/spring-test-error2.png)

比如上面报错，直接先把`AutoProxyLazyInitTests.java`和`AdvisorAutoProxyCreatorTests.java`代码注释掉。

多试几次，最后出现`BUILD SUCCESSFUL`表示成功。

## 导入到IDEA中

按如下路径选择导入：（File -> New -> Project from Existing Sources -> Navigate to directory -> Select build.gradle）

稍等一段时间，就可以确保导入成功了~

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~

> 欢迎关注我的开源项目：[一款适用于SpringBoot的轻量级HTTP调用框架](https://github.com/LianjiaTech/retrofit-spring-boot-starter)









