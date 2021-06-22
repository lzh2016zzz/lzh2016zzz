---
title: 用spring shell 写一个spring 应用监控程序
tags: java,spring,技术
---


## Spring shell介绍

不是所有应用都需要一个漂亮的用户界面,有时候很多场景也需要使用终端进行交互.

我们可以通过引入Spring Shell jar并添加自定义命令(作为spring bean上的方法)轻松构建一个功能齐全的命令行应用程序.


## Spring shell特性

Spring Shell的功能包括：

-   一个简单的，注释驱动的编程模型，用于提供自定义命令

-   使用Spring Boot自动配置功能作为命令插件策略的基础

-   选项卡完成，着色和脚本执行

-   自定义命令提示符，shell历史文件名，结果和错误的处理

-   基于域特定标准动态启用命令

-   与bean验证API集成

-   已经内置命令，如清晰的屏幕，华丽的帮助，退出

-   ASCII艺术表，带格式，对齐，花式边框等

  

## 快速完成一个Spring boot应用监控程序

现在有一个需求是 需要一个命令行应用程序,用来监控服务器上spring boot应用的状态 和 启动路径

通过`bash`脚本实现这个功能十分繁琐,但是通过`Spring shell`却可以轻而易举的做到.

完整的工程会附在文章最后

#### 1.引入依赖

首先,通过`spring initializr`初始化spring boot应用,并且引入Spring Shell依赖:

```xml
<dependency>
    <groupId>org.springframework.shell</groupId>
    <artifactId>spring-shell-starter</artifactId>
    <version>2.0.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.jetbrains.kotlin</groupId>
    <artifactId>kotlin-reflect</artifactId>
</dependency>
<dependency>
    <groupId>org.jetbrains.kotlin</groupId>
    <artifactId>kotlin-stdlib-jdk8</artifactId>
</dependency>
```

因为是用kotlin写的,所以还引用了kotlin的依赖.如果不是kotlin工程可以无视`kotlin-reflect`和`kotlin-stdlib-jdk8`

#### 2.启动应用

启动以后控制台是这样的:

相对于Spring web应用而言  `Spring Shell`多了命令提示符`shell:>`.它的作用是用来接收用户输入的指令 

![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/20200325133454.png)

Spring Shell 自带一些指令.比如`help` 显示可用命令,`clear`清屏等等.



#### 3.编写自定义指令

通过`@ShellComponent`注解可以创建自定义命令实现类

通过`@ShellMethod`注解可以实现自定义命令,方法参数即为命令参数	

现在我们需要两个命令:

* ps 

  用于显示当前服务器上运行的spring boot应用 

* kill

  用于强制中止spring boot应用
  

代码实现如下:

```kotlin
@ShellComponent("local")
class Local {

    @ShellMethod(key = ["ps"], value = "显示运行中的 spring boot")
    fun ps() {
        val data: Array<Array<String>> = localJmxConn().mapNotNull {
            it.use { conn ->
                val runtimeMXBean = conn.rmxb()
                when {
                    isSpringBootApplication(runtimeMXBean) -> {
                        val memoryMXBean = conn.mmxb()
                        val heapMemoryUsage = memoryMXBean.heapMemoryUsage
                        val heapUsage = "${heapMemoryUsage.used / (1000 * 1000)}MB/${heapMemoryUsage.max / (1000 * 1000)}MB"
                        val pid = runtimeMXBean.systemProperties["PID"] ?: ""
                        val javaVersion = runtimeMXBean.systemProperties["java.version"] ?: ""
                        val running = runtimeMXBean.systemProperties["sun.java.command"]?.
                                substring(runtimeMXBean.systemProperties["sun.java.command"]?.lastIndexOf(".")!! + 1)
                                ?: ""
                        val start = processUserFriendlyTime(System.currentTimeMillis() - runtimeMXBean.startTime)
                        val ports = getPorts(pid)
                        /**
                         * "PID","MAIN","RUNNING","HEAP","PORT","PATH","JAVA_VERSION"
                         */
                        return@mapNotNull arrayOf(pid, running, start, heapUsage, ports, StringUtils.abbreviate(runtimeMXBean.classPath, 50), javaVersion)
                    }
                    else -> return@mapNotNull null
                }
            }
        }.toTypedArray()
        printPsTable(data)
    }


    @ShellMethod(key = ["kill"], value = "kill spring boot app")
    fun kill(pid: String) {
        "kill -9 $pid".command()
        println("kill pid $pid success")
    }


}
```

尝试运行:

![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/截屏2020-03-25下午2.02.06.png)

成功.ｂ（￣▽￣）ｄ



#### 4.源码地址

[https://gitee.com/minagamiyuki/application-manager](https://gitee.com/minagamiyuki/application-manager)


