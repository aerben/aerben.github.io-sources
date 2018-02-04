title: Deploy a Java app as Docker container on AWS with Maven (Part 2)
author: Alexander Erben
date: 2017-12-02 14:07:46
tags:
- aws
- docker
desc: Part 2 describes all steps necessary to deploy docker containers from images managed in AWS ECR to EC2 instances.
---

## Introduction
In the recent [part 1](../docker-maven-1) of this series, we published a docker image containing a simple microservice to to the AWS Elastic Container Registry (ECR). 
ECR can be used like any other Docker registry to pull images and run them on any authenticated EC2 instance.

This article describes the necessary steps to provide a minimal EC2 instance that can run a docker container from an image in ECR. The instance is set up via the AWS CLI tools, and _not_ using the Web Console, on purpose. As soon as you want to go serious with your AWS endeavours, the UI won't be of much use - interaction with it cannot be feasibly automated. So, we prefer the CLI as it can be used in your setup scripts.


## Prerequisite: An existing ECR registry with images

This article assumes that you have already pushed a docker image to an ECR registry in your AWS account. You should read [part 1](../docker-maven-1) of the series, where you will create a maven project that pushes docker images to ECR. You can also read the [getting started](https://docs.aws.amazon.com/AmazonECR/latest/userguide/ECR_GetStarted.html) guide on AWS ECR and push an image to your registry.

## Provide an AWS EC2 key pair

Our docker container will be deployed on EC2, so naturally, we have to provide an instance. But there's a lot more to it: The EC2 instance must be accessible for us via SSH and HTTP, it must run the docker agent and have access to ECR where our docker images reside.

First things first: To be able to log in into our instance later on, we first have to create a key pair. If you already have a key pair registered on your AWS account, you can safely skip this section.
To create an AWS key pair for your account, follow along the instructions given by the [AWS documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-ec2-keypairs.html#creating-a-key-pair). On UNIX-like systems, the command is as follows:

```
$ aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > MyKeyPair.pem
$ chmod 400 MyKeyPair.pem
```
Remember the location where you stored the key file. You will need it later.

## Create a security group

By default, any EC2 instance you create won't be accessible from your local machine. We will have to make sure you can log in to the instance via SSH to install docker and run images. Access must be granted over port 22 for SSH and, in case you use the docker image created in the [last article](../docker-maven-1), port 8080 for the web application. 

This article will not go into great detail on AWS networking. The instance as well as all associated networking resources will be deployed in the account's default VPC. If you don't yet know what a VPC is, that's fine - think of it as an isolated network in the AWS cloud where you can safely run your EC2 instances.

To open up the necessary ports for access from your local machine, we will use security groups. A security group in AWS behaves like a firewall preventing or granting access to your instances to a range of source IP addresses, protocols and ports. In our example, we need TCP over port 22 for SSH and TCP over port 8080 for HTTP from the IP address of your computer - or, if you don't know your IP address and are lazy, for the whole world.

To create a security group, use the following command:

```
$ aws ec2 create-security-group --group-name DockerHostSecurityGroup --description "A security group for the docker tutorial on aerben.me"
```

Then go ahead and allow access for ports 22 and 8080 over TCP. The given commands will allow access to every source IP address by specifying the CIDR block  `0.0.0.0/0`. If you know your public IP address or a range in which it will be, you can and should substitute the appropriate CIDR range.

```
$ aws ec2 authorize-security-group-ingress --group-name DockerHostSecurityGroup --protocol tcp --port 22 --cidr "0.0.0.0/0"
$ aws ec2 authorize-security-group-ingress --group-name DockerHostSecurityGroup --protocol tcp --port 8080 --cidr "0.0.0.0/0"
```

That's all. We don't have to explicitly set egress ("outbound") rules because security groups in AWS are _stateful_. In simple terms, that means that all traffic that has been allowed in is also allowed back out. 

The security group is now ready to use for the EC2 instance.

## Run an EC2 instance

Now that we have all prerequisites in place, actually creating the instance is pleasantly simple. 

All you need is the key and the security group you just created and a so-called Amazon Machine Image (AMI) ID. This is the image used to create the boot volume of your instance. The example below uses the AMI for Amazon Linux in the EU (Frankfurt) region. You can look up the Amazon Machine Image for your region [here](https://aws.amazon.com/de/amazon-linux-ami/). Be sure to use the "HVM (SSD) EBS-backed 64 Bit" AMI.
The command to run a free-tier eligible instance is as follows:

```
$ aws ec2 run-instances --image-id ami-5652ce39 --count 1 --instance-type t2.micro --key-name MyKeyPair --security-groups DockerHostSecurityGroup
```

We will now have to wait until the instance comes alive. The following command will tell you the current state of your instance as well as its public IP address and Instance ID, both of which we will soon need:

```
$ aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,PublicIpAddress,State.Name]"
[
    [
        [
            "i-0c701dac19ec55138",
            "54.93.100.114",
            "running"
        ]
    ]
]
```

As soon as the EC2 instance state switches to "running", you can continue with the tutorial. 

## Connect to the instance and install docker

The EC2 instance is now running, so we should now go ahead and connect to it via SSH. Note that the "running" state of an EC2 instance just means that the virtual machine has started - the OS itself might still be booting up. That means that you might still have to wait some minutes to be able to connect to the server.

The default user for Amazon Linux is `ec2-user`, so given the private key file you created above and the public IP address of your instance, you can open up the connection as follows:

```
$ ssh -i MyKeyPair.pem ec2-user@35.158.97.195
  The authenticity of host '35.158.97.195 (35.158.97.195)' can't be established.
  ECDSA key fingerprint is SHA256:UJlSED90KpQRAwc1QMATbj/RFjQZ5DZRsyY65Lt9czE.
  Are you sure you want to continue connecting (yes/no)? 
  
  yes
  
  Warning: Permanently added '35.158.97.195' (ECDSA) to the list of known hosts.
  
         __|  __|_  )
         _|  (     /   Amazon Linux AMI
        ___|\___|___|
  
  https://aws.amazon.com/amazon-linux-ami/2017.09-release-notes/
  1 package(s) needed for security, out of 1 available
  Run "sudo yum update" to apply all updates.
```

You should now run `sudo yum update -y` to install all updates.
Next, we need to install and start docker on our EC2 instance. Using yum, this isn't hard at all:

```
$ sudo yum install -y docker
$ sudo service docker start
```

This will install docker on your machine. Now, the last thing we have to do is download and run the image and were done... right?
It's not that simple. If we try to deploy the image via ECR (remember to replace the account id), the following will happen:

```
$ sudo docker run -p 8080:8080 [[ACCOUNT_ID]].dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service:1.0

Unable to find image '427866372521.dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service:1.0' locally
docker: Error response from daemon: Get https://427866372521.dkr.ecr.eu-central-1.amazonaws.com/v2/spark-sample-service/manifests/1.0: no basic auth credentials.
See 'docker run --help'.
```

Oh yeah, we've seen that already in the last article. We need to perform a docker login against ECR to pull an image.

## Authenticate docker with ECR and run the image

If you have read the last article, this is no news for you: for the instance to gain access to ECR, you must first authenticate docker against the registry.
To that end, use the AWS ECR tools to retrieve credentials for logging in. The `get-login` command returns a ready-to-use command for `docker login` to authenticate.

```
$ aws ecr get-login --no-include-email
docker login -u AWS -p PASSWORD -e none https://[[ACCOUNT_ID]].dkr.ecr.eu-central-1.amazonaws.com
```

Then at last, start a container from your image

```
$ sudo docker run -p 8080:8080 [[ACCOUNT_ID]].dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service:1.0`

Unable to find image '427866372521.dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service:1.0' locally
1.0: Pulling from spark-sample-service
ff3a5c916c92: Pull complete
5de5f69f42d7: Pull complete
fa7536dd895a: Pull complete
fa92dd14eb3f: Pull complete
be90dab9ef0a: Pull complete
Digest: sha256:969cef8230f2bf5dd1c03a65c0af1372e7c81930c1c26caa70027dda80ec605f
Status: Downloaded newer image for 427866372521.dkr.ecr.eu-central-1.amazonaws.com/spark-sample-service:1.0
[Thread-0] INFO org.eclipse.jetty.util.log - Logging initialized @433ms
[Thread-0] INFO spark.embeddedserver.jetty.EmbeddedJettyServer - == Spark has ignited ...
[Thread-0] INFO spark.embeddedserver.jetty.EmbeddedJettyServer - >> Listening on 0.0.0.0:8080
[Thread-0] INFO org.eclipse.jetty.server.Server - jetty-9.3.6.v20151106
[Thread-0] INFO org.eclipse.jetty.server.ServerConnector - Started ServerConnector@46a08f0f{HTTP/1.1,[http/1.1]}{0.0.0.0:8080}
[Thread-0] INFO org.eclipse.jetty.server.Server - Started @650ms
```

and verify your setup by calling your service via the instance's public IP address:

```
curl 35.158.97.195:8080
It's me!
```

And that's it!

Don't forget to terminate your instance as soon as you are done:

```
aws ec2 terminate-instances  --instance-ids i-0408fef6a295da99e
```

Replace the instance id with the one of your own instance.

## Conclusion

Over the course of the last two articles, we've learned a lot:

- Packaging a Java application as docker image with the Spotify Dockerfile Maven Plugin
- Authenticate Docker against the AWS Elastic Container Registry (ECR)
- Pulling and running images from ECR
- Creating AWS EC2 key pairs
- Granting network access to EC2 instances via security groups
- Starting and terminating EC2 instances via the command line

I hope you've enjoyed the series and am glad to hear your feedback in the comments section below!
 

