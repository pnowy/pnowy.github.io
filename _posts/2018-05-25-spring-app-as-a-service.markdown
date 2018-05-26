---
layout: post
comments: true
title:  "How to install spring boot app as a service and replace version by scp"
description: "How to install spring boot app as a service and replace version by scp"
date:   2018-05-25
categories: Spring
tags: java spring linux scp
---

In the time of microservices and Kubernetes most of the java applications are deployed
as Docker containers (Kubernetes pods) and old fashioned deployment to the application
server or as a operating system service is rare but sometimes useful.

I will show how to simply install Spring boot application as a linux service
and deploy the newer application version by the simple script.

At the beginning let's assume we have ready Spring Boot application packaged as JAR file with embedded
web server. Next to the JAR file we could put `application.properties` or `application-{profile}.properties`
in order to parametrize application (see more about [externalized configuration on Spring Boot documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-external-config).

In this post I will show how to run Spring Boot app by the `systemd`. `systemd` is the successor of the 
System V init system and is now being used by many modern Linux distributions.

First of all create a service file in `/etc/systemd/system` named e.g. myapp.service with the following content:

```
[Unit]
Description=myapp
After=syslog.target

[Service]
User=root
WorkingDirectory=/var/app
ExecStart=/root/.sdkman/candidates/java/current/bin/java -jar /var/app/myapp.war
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

Remember to change the _Description_, _User_, and _ExecStart_ fields for your application.
In the above case the working directory is set to `/var/app` and the service is run by the root.
It is not always the best solution from security perspective - you should create separate user
account with proper permissions. You could also notice that _ExecStart_ run normal java process
and points to the generated Spring Boot application package. Because the java is not added to the path
(you could use tools like [SDKMAN](http://sdkman.io/index.html)) exec has full path to the java binary.

With the deployed app and prepared script notify `systemd` about the service:

```
systemctl daemon-reload
```

And enable service:
 
 ```
systemctl enable myapp.service
```

You could control new service with the give commands:

```
systemctl start myapp
systemctl stop myapp
systemctl restart myapp
systemctl status myapp
```

When we configure the server side we need a Bash script  

```
#!/usr/bin/env bash

# configuration
SERVER="ubuntu@111.111.222.222"
KEY="~/.ssh/demo-server.pem"
APP_NAME=myapp

# prepare new jar/war (this is the old version of spring, with 2.X ./gradlew clean build)
./gradlew clean bootRepackage 
# stop remote service
ssh -i $KEY $SERVER "sudo systemctl stop $APP_NAME"
# move old var (backup old versions and logs - modify if needed)
ssh -i $KEY $SERVER "cp /var/app/$APP_NAME.war /var/app/\$(date +%F-%H:%M)_$APP_NAME.war && cp /var/app/$APP_NAME.log /var/app/\$(date +%F-%H:%M)_$APP_NAME.log"
# copy new app version to remote server
scp -i $KEY ./build/libs/*.war $SERVER:/var/app/$APP_NAME.war
# start remote service
ssh -i $KEY $SERVER "sudo systemctl start $APP_NAME"
```

It's a general script which helps you to build, stop remote service, copy current app version and start new service. It should be enough when you have some
kind of demo server and need fast option to redeploy application.