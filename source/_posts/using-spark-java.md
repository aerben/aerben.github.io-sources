---
title: Using Spark as an alternative to Spring Boot
date: 2017-11-11 10:00:00
tags: java
desc: There are many options for writing Java Microservices. Here, I will start to explore the most minimal of possible approaches - the Spark Framework.
---

There are many options for writing Java Microservices. Here, I will start to explore the most minimal of possible approaches - the Spark Framework.

## Motivation

For a long time now, Java Microservices have been equivalent with Spring Boot for me. It combines an almost ridiculously easy setup - at least for Java standards - with the unparalleled feature-richness of the Spring Framework.
However, Spring Boot has it's downsides. As the dependency tree of your application grows, it is crucial to constantly check if unwanted auto-configuration is being bootstrapped. Spring is king in hiding away abstractions from the user, to a point where it becomes almost impossible to debug issues in auto-configuration classes without in-depth knowledge of the framework. This is especially a problem when using Spring Cloud with its numerous sometimes under-documented features and configuration properties.

Sometimes, it is relieving to tear away everything that makes Java services so cumbersome and heavyweight and start with something really nice and easy. Spark Framework is an example for a bare-bone approach to Java Microservices with a design that resembles node.js web frameworks like express.

## Basic setup

Let's start with the `pom.xml`. In the first iteration, we do want to bother with making the project executable as a jar file, nor do we want a proper logging setup.

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>me.aerben</groupId>
    <artifactId>sample-service</artifactId>
    <version>1.0</version>
    <name>sample-service</name>
    <dependencies>
        <dependency>
            <groupId>com.sparkjava</groupId>
            <artifactId>spark-core</artifactId>
            <version>2.5</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

This is already enough to pull together a basic Spark project with Java 8 support for concise code. Now we just need the actual main application class (as always with Maven, it goes into `/src/main/{package}`) and we're good to go:

```java
package me.aerben;

import static spark.Spark.*;

public class Main {
    public static void main(String[] args) {
        port(8080);
        get("/", (req, res) -> "It's me!");
    }
}

```

Being a loyal Tomcat user for a long time, I just had to change the listen port from its default `4567` to `8080`, but that is of course entirely up to you. We can now start the application out of an IDE like Eclipse or IntelliJ IDE and then query it to receive the expected result.

```
$ curl localhost:8080
It's me!
```

## Building a standalone package

Now we've got something that we can run in our IDE. But when we try to build the app and run it on the command line, it will not work:
```
$ mvn package
$ java -jar target/sample-service-1.0.jar
no main manifest attribute, in "sample-service-1.0.jar"
```
Well, we did not configure our jar build yet. To fix this, we have to apply some tweaks to the project's `pom.xml`. First, we add the `maven-jar-plugin` to the project's build configuration so that an executable jar is built:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>2.4</version>
    <configuration>
        <finalName>${project.name}</finalName>
        <archive>
            <manifest>
                <addClasspath>true</addClasspath>
                <mainClass>me.aerben.Main</mainClass>
                <classpathPrefix>dependency-jars/</classpathPrefix>
            </manifest>
        </archive>
    </configuration>
</plugin>
```

When we build the project with `mvn package` now, we obtain a jar with the project's name in the target folder.
However, it still won't run:

```
$ java -jar target/sample-service.jar

Exception in thread "main" java.lang.NoClassDefFoundError: spark/Spark
	at me.aerben.Main.main(Main.java:7)
Caused by: java.lang.ClassNotFoundException: spark.Spark
	at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
```

The dependencies are still absent from the build jar. We have to include another plugin to bundle a jar that contains everything that is necessary to start the application.


```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>attached</goal>
            </goals>
            <phase>package</phase>
            <configuration>
                <finalName>${project.name}</finalName>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
                <archive>
                    <manifest>
                        <mainClass>me.aerben.Main</mainClass>
                    </manifest>
                </archive>
                <!--<appendAssemblyId>false</appendAssemblyId>-->
            </configuration>
        </execution>
    </executions>
</plugin>
```
The assembly plugin will generate a jar file with all dependencies bundled in the target folder. The name of the file is `sample-service-jar-with-dependencies.jar` in our case.
We can now run the app as a normal java application:

```
$ java -jar target/sample-service-jar-with-dependencies.jar
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
```

```
$ curl localhost:8080
It's me!
```
## Final tweaks

Everything is working fine - but what to do against the ugly SLF4J-errors? It turns out that Spark adds Slf4J as logging facade to the classpath, but thankfully lets us choose the implementation we want to use. The slf4j-simple implementation will just dump the logs to standard output and required no additional configuration.

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-simple</artifactId>
    <version>1.7.25</version>
</dependency>
```

Another thing we might want to change is the file name of the generated jar with dependencies. If we want to get rid of the assembly identifier in the file name, we can add the configuration property `appendAssemblyId` which you can see commented out in the above configuration of the `maven-assembly-plugin`. Note that when you use the same finalName in the maven-assembly-plugin than you use in the maven-jar-plugin, it will result in a warning that the original jar file is being overwritten.

Great, so now we have a running Spark app that can be started directly from the command line. In the next post, we want to further explore what to do with it.