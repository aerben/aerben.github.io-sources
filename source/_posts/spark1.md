---
title: Using Spark as an alternative to Spring Boot (part 1)
date: 2017-11-11 10:00:00
tags: java
---

There are many options for writing Java Microservices. Here, I will start to explore the most minimal of possible approaches - the Spark Framework.

For a long time now, Java Microservices have been equivalent with Spring Boot for me. It combines an almost ridiculously easy setup - at least for Java standards - with the unparalleled feature-richness of the Spring Framework.
However, Spring Boot has it's downsides. As the dependency tree of your application grows, it is crucial to constantly check if unwanted auto-configuration is being bootstrapped. Spring is king in hiding away abstractions from the user, to a point where it becomes almost impossible to debug issues in auto-configuration classes without in-depth knowledge of the framework. This is especially a problem when using Spring Cloud with its numerous sometimes under-documented features and configuration properties.

Sometimes, it is relieving to tear away everything that makes Java services so cumbersome and heavyweight and start with something really nice and easy. Spark Framework is an example for a bare-bone approach to Java Microservices with a design that resembles node.js web frameworks like express.

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

As a Tomcat veteran I just had to change the listen port from its default `4567` to `8080`, but that is of course entirely up to you. We can now start the application out of an IDE like Eclipse or IntelliJ IDE and then query it to receive the expected result.

```
$ curl localhost:8080
It's me!
```

Now we've got something that we can run in our IDE. But when we try to build the app with `mvn package` and then run it with `java -jar target/sample-service-1.0.jar`, the java command line runner informs us that the main manifest attribute is missing. Well, we did not configure our jar build yet and it won't be enough just to include the maven-jar-plugin in the build lifecycle. So let's explore this further in the next article.