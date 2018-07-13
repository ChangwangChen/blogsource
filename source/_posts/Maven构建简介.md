---
title: Maven构建简介
date: 2018-07-12 16:43:34
categories: Java
tags: [Java, Maven]
---
### 打包

今天自己完全脱离框架创建了一个 `Maven` 项目, 在构建生成 Jar 包并且启动的时候报错.

<!--more-->
首先出现的错误如下:

```xml
<!-- 报错信息: no main manifest attribute, in qps-1.0-SNAPSHOT.jar -->

<!-- 而此时的 pom.xml 中 <build> 的信息如下 -->

<build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
</build>

```

很明显, 没有指定 `jar` 报中程序执行的入口文件. 于是, 我在 <build> 中添加了下一个 <plugin>:

```xml

   <plugin>
      <!-- Build an executable JAR -->
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-jar-plugin</artifactId>
      <version>3.1.0</version>
      <configuration>
        <archive>
          <manifest>
            <addClasspath>true</addClasspath>
            <classpathPrefix>lib/</classpathPrefix>
            <mainClass>com.mypackage.MyClass</mainClass>
          </manifest>
        </archive>
      </configuration>
    </plugin>

```

这时, 再去执行 `java -jar` 命令的时候, 又出现了新的错误.

```xml

<!-- 错误: Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/commons/lang3/StringUtils
 -->

 <!-- 说明项目里面引用的第三方的包没有一起放进 jar 包里面 -->

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>2.4.1</version>
                <configuration>
                    <archive>
                        <manifest>
                            <!-- 入口文件 -->
                            <mainClass>com.mypackage.MyClass</mainClass>
                        </manifest>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
    
    <!-- 加入上面的 plugin 之后, 终于是可以运行了 -->

```

加上上述指令之后, maven 生成的 jar 包总算是可以直接使用 `java -jar` 指令来运行了.

但是以上的指令生成的 jar 包会有一下问题:

* 配置文件也在 jar 文件里面, 想要修改配置的话, 需要重新打包部署
* jar 包的体积过大, 传输不方便

于是, 我们可以只打包自己的代码生成 jar 包, 在使用

```sh

mvn dependency:copy-dependencies -DincludeScope=compile -DoutputDirectory=/lib

```

将引用的第三方 jar 包都复制到指定的目录中. 再将我们的代码也打包之后复制进 lib 目录中, 最后使用 

```bash

nohup java -Xms512m -Xmx512m -Djava.ext.dirs=lib/ -cp . your.java.main.classname  &

```

指令直接执行即可.


### 读取资源文件

我们使用 Maven 来构建 Java 项目的时候, 约定的目录结构如下:

```
 --- src/main/
         |
         --- java/ : 代码目录
         --- resource/ : 资源目录
             |
             --- config.properties
 --- target/ : maven 生成信息目录

```

Java 代码的运行是要先进行编译的, Maven 编译过后会将代码和资源文件都复制进 `target/classes` 目录中, 所以我们在代码中读取资源文件的时候, 需要考虑的是编译过后的文件目录. 直接看代码:

```java

    //保存配置信息
    private prproperties = new Properties();

    //参数是配置文件的路径
	private void loadProperties(String path) {
        // 首先读取的是 jar 中的配置文件
        // 编译过后, 没有生成 jar 文件之前, 就是 target/classes 目录
		try(InputStream in = this.class.getResourceAsStream(path)) {
			this.properties.load(in);
		} catch (Exception e) {
			e.printStackTrace();
		}

        // "user.dir" 对应的是程序启动的文件目录
        // 我们在部署的时候, 会将配置文件复制到指定文件目录下面的 resources 目录里面
		String filePath = System.getProperty("user.dir") + "/resources/" + path;
		if(new File(filePath).exists()){
			InputStream in;
			try {
				in = new BufferedInputStream(new FileInputStream(filePath));
				this.properties.load(in);
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}

```

### 就写到这里吧