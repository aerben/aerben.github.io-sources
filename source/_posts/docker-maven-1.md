---
title: Deploy a Java app as Docker container with Maven on AWS (part 1)
date: 2017-11-19 14:01:46
tags: 
- java
- aws
desc: Deploying docker containers to AWS is fun and there are many options. Here, I explore a simple approach for Java applications that uses Spotify's dockerfile-maven plugin.
---

## Introduction
There are numerous ways to package a Java application into a Docker image and then deploy it to AWS. The usual approach would be to set up an [ECS](https://aws.amazon.com/ecs/) cluster to deploy containers. Actually, I will write a post over benefits and pitfalls of using ECS later on. For now, we will do the bare-bones approach and use a simple EC2 instance set up via Cloudformation to deploy a docker container.

To follow along, you can have a look at the source code of the resulting setup I created on [{% fa github %} GitHub](https://github.com/aerben/aerben.github.io-samples/tree/master/docker-maven). However, if you want to try things out yourself, you can start with your own maven based application or use my sample application that I mention below.

## Preparations
In a [recent post](../using-spark-java), I created a minimal Spark Java application built with Maven which we can take as baseline. You can get the source code from my [{% fa github %} GitHub](https://github.com/aerben/aerben.github.io-samples/tree/master/spark-sample-service) repository.

After cloning the repo, you can build and run the app as follows
```
$ mvn package && java -jar target/spark-sample-service-jar-with-dependencies.jar
[Thread-0] INFO org.eclipse.jetty.server.ServerConnector - Started ServerConnector@1b7eb264{HTTP/1.1,[http/1.1]}{0.0.0.0:8080}
[Thread-0] INFO org.eclipse.jetty.server.Server - Started @326ms
```
The app now listens to port 8080 and sends you a neat little message when you point your browser to it. 

We now have an application we want to deploy, the next question is how we want to build a docker image of it. It's super easy to just write a simple Dockerfile using the Java 8-Alpine base image, adding the jar and executing it in the entrypoint. 

This time however, we want to fully integrate the creation of a Docker image into the Maven build lifecycle. To that end, we use Spotify's [dockerfile-maven-plugin](https://github.com/spotify/dockerfile-maven). It not only streamlines our build automation pipeline by reducing the number of steps necessary to perform a build and deployment. It also speeds up the Docker image build by caching Maven dependencies. At least that's what Spotify promises.  

## Creating an ECR repository

First things first: As we want to deploy our application into AWS, we want to set up a Container Registry in our AWS account. Amazon Web Services' offering to create a registry is called Elastic Container Registry (ECR).
I assume you have already set up the [AWS CLI](https://aws.amazon.com/cli‎) on your machine, and have attached the [necessary permissions](http://docs.aws.amazon.com/AmazonECR/latest/userguide/ECR_IAM_policies.html) to create an ECR-Repository and push images to it. We can now create a new ECR registry as follows:

```
$ aws ecr create-repository --repository-name spark-sample-service

{
    "repository": {
        "registryId": "REPO_ID",
        "repositoryName": "spark-sample-service",
        "repositoryArn": "arn:aws:ecr:eu-central-1:REPO_ID:repository/spark-sample-service",
        "createdAt": 1511103918.0,
        "repositoryUri": "REPO_ID.dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service"
    }
}
```

Take note of the repository ARN and URI for later reference so you don't have to look it up in the AWS Console later on. That is all regarding ECR for now.

## Using dockerfile-maven

Parts of the following description is based off the samples provided by Spotify for the dockerfile-maven-plugin, especially the [advanced](https://github.com/spotify/dockerfile-maven/tree/master/plugin/src/it/advanced) example.
The first thing we have to make sure is that all project dependencies are copied into a separate folder in the project's target directory. This is a notable difference from the [sample application](https://github.com/aerben/aerben.github.io-samples/tree/master/spark-sample-service) that I have provided above, which bundles all dependencies into one fat jar that includes everything to be executable. Spotify recommends the former approach as it allows to cache individual dependencies in the docker image. 

Remove the maven-assembly-plugin and maven-jar-plugin entries from the pom of the sample application and add the following plugins to the build-section of the pom.xml:

```xml
<plugin>
    <artifactId>maven-jar-plugin</artifactId>
    <configuration>
        <archive>
            <manifest>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
                <mainClass>me.aerben.Main</mainClass>
            </manifest>
        </archive>
    </configuration>
</plugin>
```

```xml
<plugin>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <phase>initialize</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <overWriteReleases>false</overWriteReleases>
                <includeScope>runtime</includeScope>
                <outputDirectory>${project.build.directory}/lib</outputDirectory>
            </configuration>f
        </execution>
    </executions>
</plugin>
````

The maven-jar-plugin configuration makes sure that the source code of your application is bundled with a fitting manifest file that specifies the main class and classpath entries necessary to find the dependencies relative to the jar file location. The dependency plugin then makes sure that all dependencies are copied into the `lib` folder in the project's target.

We will now configure the dockerfile-plugin itself.
```xml
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>dockerfile-maven-plugin</artifactId>
    <version>1.3.6</version>
    <executions>
        <execution>
            <id>default</id>
            <goals>
                <goal>build</goal>
                <goal>push</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <repository>REPO_ID.dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service</repository>
        <tag>${project.version}</tag>
        <buildArgs>
            <JAR_FILE>${project.build.finalName}.jar</JAR_FILE>
        </buildArgs>
    </configuration>
</plugin>

```
Remember that you have to put the repository URI of your ECR registry in the configuration. The rest is straightforward: the plugin will hook into Maven's build lifecycle and build the image when the application is packaged, then push it when it is deployed.

The last missing thing is a Dockerfile that specifies how our docker image is to be built.
```dockerfile
FROM openjdk:8-jre-alpine

ENTRYPOINT ["/usr/bin/java", "-jar", "/tmp/app/app.jar"]

ADD target/lib           /tmp/app/lib
ARG JAR_FILE
ADD target/${JAR_FILE} /tmp/app/app.jar
```
As you can see the jar _and_ the library dependencies are added to the image, allowing the plugin to cache individual dependencies and not always copy a fat jar on each build,

We can now try our setup by running `mvn deploy`:

```
$ mvn deploy
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy (default-deploy) on project spark-sample-service: Deployment failed: repository element was not specified in the POM inside distributionManagement element or in -DaltDeploymentRepository=id::layout::url parameter -> [Help 1]
```
Something's missing: When running `mvn deploy`, by default Maven wants to push the built artifact into an artifact manager like Nexus. To that end, the repository must be configured in the distribution management of the pom. If you do not have an artifact manager at hand, you can disable this behaviour by adding the following build plugin configuration:
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-deploy-plugin</artifactId>
    <configuration>
        <skip>true</skip>
    </configuration>
</plugin>
```

## Authenticate Docker against ECR
We can now try again:
```
$ mvn package
[ERROR] Failed to execute goal com.spotify:dockerfile-maven-plugin:1.3.6:push (default) on project spark-sample-service: Could not push image: no basic auth credentials -> [Help 1]
```
This starts to get annoying! But the mistake is on our side: In order to push images to ECR, we have to authenticate against it with basic auth credentials. Luckily, this is a very easy task with the help of the AWS CLI.
```
$ aws ecr get-login
docker login -u AWS -p PASSWORD -e none https://REPO_ID.dkr.ecr.eu-central-1.amazonaws.com
```
The cli returns for us a ready-to use command line call to authenticate our local Docker installation with ECR. 
Well, almost ready-to-use. Because on newer versions of Docker, you have to remove the `-e` flag, which makes automation a bit harder.

Anyway, after having logged in, we could assume that the plugin picks up the credentials and uses it to authenticate against ECR. On Linux, this will work, but sadly, on macOS, Docker by default uses the macOS keychain to store the credentials (you can see it in `~/.docker/config.json`), and this workflow is [not working with dockerfile-maven](https://github.com/spotify/docker-maven-plugin/issues/321).

To work around this issue, we can store the credentials in `.m2/settings.xml` and tell the plugin to use these credentials to authenticate against ECR. This is an inconvenient approach as it makes automation much harder: the credentials are only temporary, so we have to write a script that puts the credentials in `settings.xml` everytime they expire.
Anyway: If you a are a Mac user like me, put the folling server entry in `.m2/settings.xml`:
```xml
<server>
    <id>REPO_ID.dkr.ecr.eu-central-1.amazonaws.com</id>
    <username>AWS</username>
    <password>[YOUR_PASSWORD]</password>
</server>
```
Please don't forget to fill in your ECR repository ID and temporary password.
Then go ahead and add the `useMavenSettingsForAuth` configuration property to the dockerfile-maven-plugin declaration:

```xml
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>dockerfile-maven-plugin</artifactId>
    (...)
    <configuration>
        <repository>REPO_ID.dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service</repository>
        (...)
        <!-- the following one -->
        <useMavenSettingsForAuth>true</useMavenSettingsForAuth>
    </configuration>
</plugin>

```
This tells the plugin to take the credentials from `settings.xml`.
Now, finally, we can push the image:
```
$ mvn deploy
(...)
[INFO] --- dockerfile-maven-plugin:1.3.6:push (default) @ spark-sample-service ---
[INFO] The push refers to a repository [REPO_ID.dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service]
[INFO] Image 11f5b463d7c9: Preparing
[INFO] Image 6c374bfe7f7e: Preparing
[INFO] Image 6abaa286b4af: Preparing
[INFO] Image 7f22a835d8ba: Preparing
[INFO] Image 2aebd096e0e2: Preparing
[INFO] Image 2aebd096e0e2: Layer already exists
[INFO] Image 7f22a835d8ba: Layer already exists
[INFO] Image 6abaa286b4af: Layer already exists
[INFO] Image 6c374bfe7f7e: Layer already exists
[INFO] Image 11f5b463d7c9: Pushing
[INFO] Image 11f5b463d7c9: Pushed
[INFO] 1.0: digest: sha256:b4823b325f0ef9c016bcba35024486a1bcd80f1b2d69315904c03521fee63aa8 size: 1366
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 11.147 s
[INFO] Finished at: 2017-11-19T16:54:46+01:00
[INFO] Final Memory: 29M/458M
[INFO] ------------------------------------------------------------------------
```

And to test it, we can simply start a container from the newly pushed image:
```
$ docker run -p 8080:8080 REPO_ID.dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service:1.0 &
$ curl localhost:8080
It's me!
```

Whew, that was a lot of work - and the EC2 part has not even begun. Let's take a break, grab a coffee and come back later in another post.

## References
+ Spotify's dockerfile-maven-plugin: https://github.com/spotify/docker-maven-plugin
+ ECR reference documentation: https://aws.amazon.com/documentation/ecr/