---
title: Using Spark as an alternative to Spring Boot (part 2)
date: 2017-11-18 18:25:26
tags: java
---

In the [previous article](/2017/11/11/spark1), we've set up a basic Spark application that is not yet able to run standalone as a jar application. To achieve this, we have to apply some tweaks to the project's `pom.xml`.
First, we add the `Maven Jar Plugin` to the project's build configuration so that an executable jar is built:

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

Our configuration starts to get bulky - as it always does when Maven projects become actually useful. The assembly plugin will generate a jar file with all dependencies bundled in the target folder. The name of the file is `sample-service-jar-with-dependencies.jar` in our case. If we want to get rid of the assembly identifier in the file name, we can add the configuration property                 ```<appendAssemblyId>false</appendAssemblyId>```. Note that in the above example, this will result in a warning that the original jar file is being overwritten. To prevent is, you can apply another finalName to the jar generated in the assembly.

We can now run the app as a normal java application:

```
java -jar target/sample-service-jar-with-dependencies.jar
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
```

```
$ curl localhost:8080
It's me!
```

Everything is working fine - but what to do against the ugly SLF4J-errors? 
We will fix this issue in the next post - and further explore what we can do with our new application.