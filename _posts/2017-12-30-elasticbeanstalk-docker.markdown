---
layout: post
comments: true
title:  "AWS Elastic Beanstalk docker application deployment and CI/CD Gitlab configuration"
description: "AWS Elastic Beanstalk docker application deployment and CI/CD Gitlab configuration"
date:   2017-12-30
categories: DevOps
tags: aws docker gitlab ci cd
---

In this post I will show how to prepare and execute AWS elastic beanstalk deployment of docker application. I think these days docker deployment 
is the best way to package and deploy most of the web applications.
 
Recently I had deployed two totally different applications - one Python Django project and one Java Spring Boot application - both the same very
easy and efficient way - by the Docker containers.

There is a couple of options when you deploy Docker containers to Amazon Elastic Beanstalk. For simple applications there is **Single Container Docker**
which could be build from Dockerfile for us by Amazon. For complex and performance demanding applications exist **Multicontainer Docker** option.
In this case we have to use prebuild Docker images from our public or private registry. In this tutorial I will describe both ways. There is also third
option where we could use preconfigured images but I won't describe this path.

So let's assume that we have simple [Django/Python application (I've created something on my Github account)](https://github.com/pnowy/elections) 
and we would like to create simple flow that when we push the code to the master branch this code will be deployed to the our Elastic Beanstalk
environment - just for test I do not distinguish _dev_, _prod_, etc. - it all could by done just by proper **.gitlab-ci.yml** file reconfiguration.

First of all we have to prepare the Dockerfile and put this file on the root of our project. AWS will use it to build our Docker image. I've prepared
the simplest example of that kind of Dockerfile as possible (of course for production ready Django app we need to extend this file but this is not
the intention of this post):

```
FROM python:3.6.3
MAINTAINER Przemek Nowak

COPY . /mysite
WORKDIR /mysite

RUN python -m pip install -r requirements.txt

EXPOSE 8000

CMD ["sh", "./runserver.sh"]
```

The content of the `runserver.sh` just starts the server:

```bash
python manage.py runserver 0.0.0.0:8000
```

For **Single Container Docker** deployment we don't need `Dockerrun.aws.json` - alone Dockerfile is enough to deploy and run our application on 
AWS Elastic Beanstalk. In case of **Multicontainer Docker** we have to prepare the Docker image first (we could build image in our Gitlab server
to available registry - I will show example later) and then declare JSON file on the project root directory: 

```json
{
    "AWSEBDockerrunVersion": "1",
    "Authentication": {
        "Bucket": "mysite-bucket-docker-auth",
        "Key": "dockercfg"
    },
    "Image": {
        "Name": "registry.gitlab.com/mycompany/mysite",
        "Update": "true"
    },
    "Ports": [
        {
            "ContainerPort": "8000"
        }
    ],
    "Logging": "/var/log/nginx"
}
```

- _Authentication_ - this part is required only for private Docker repositories, we have to specify the Amazon S3 object storing the `.dockercfg` file  
- _Image_ - specifies the Docker base image on an existing Docker repository from which you're building a Docker container  
- _Ports_ - when we specify prepared earlier image we have to point the list of ports to expose on the Docker container. These port will be connected with the reverse proxy running on the host   

In both cases when we prepared `Dockerfile` or `Dockerrun.aws.json` we have to create Elastic Beanstalk environments. We could automate this
process if we need for example new environment for each feature branch or something like that but here I will show how we could environment manually
by AWS EB CLI. The details how to install and configure it you will find on [AWS documentation](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html)
so I will skip this part.

In order to create new environment you have to execute `eb create` command available on AWS EB CLI. When you do that you have to fill some fields required 
to setup the new environment.
  
```text
$ eb create
Enter Environment Name
(default is tmp-dev): ENTER
Enter DNS CNAME prefix
(default is tmp-dev): ENTER
Select a load balancer type
1) classic
2) application
3) network
(default is 1): ENTER
NOTE: The current directory does not contain any source code. Elastic Beanstalk is launching the sample application instead.
Do you want to download the sample application into the current directory?
(Y/n): ENTER
INFO: Downloading sample application to the current directory.
INFO: Download complete.
Environment details for: tmp-dev
  Application name: tmp
  Region: us-east-2
  Deployed Version: Sample Application
  Environment ID: e-um3yfrzq22
  Platform: 64bit Amazon Linux 2014.09 v1.0.9 running PHP 5.5
  Tier: WebServer-Standard-1.0
  CNAME: tmp-dev.elasticbeanstalk.com
  Updated: 2017-11-08 21:54:51.063000+00:00
Printing Status:
INFO: createEnvironment is starting.
 -- Events -- (safe to Ctrl+C) Use "eb abort" to cancel the command.
```

As can you see on example from AWS documentation you have to provide the following information:

- _Environment Name_ - the name of new created environment
- _DNS CNAME prefix_ - the DNS for our environment, it has to be unique
- _Load Balancer Type_ - you have to select load balancing type for you application, the detailed comparison is available on 
[AWS Documentation](https://aws.amazon.com/elasticloadbalancing/details/#compare), for testing purposes the classic LB is ok.
- _Application code_ - in our case with prepared Docker configuration and application code EB CLI should pack and deploy source code from current branch.
Take into account that by default EB CLI takes only the code which is committed to the version control system.

After create operation the EB CLI will create the `.elasticbeanstalk` directory with the `config.yml` file within it. You could commit this config
file to VCS but don't keep any secrets and similar data there. Everything what should be secret cannot be on the VCS.

In the next step we will prepare the `.gitlab-ci.yml` file where we define the job to deploy latest code for our environment. Example file:

```
image: python:3.6.3

stages:
  - deploy

deploy-staging:
    stage: deploy
    script:
      - python3 -m pip install -r requirements.txt
      - eb use staging-environment -v
      - eb deploy staging-environment -v
    environment:
      name: staging
      url: http://my.staging.environment
    only:
      - develop
```

Of course the example file is very simple and has only one stage (**deploy**). Normally this file could have a lot of stages (test, quality, package, etc.)
and deploy to many environments. The script is for Python application that's why all dependencies are installed - the awsebcli package is defined
on the `requirements.txt` file. You could also notice that this deployment is run only for develop branch. If you would like to read more about
possibilites of GitLab CI you could take a look [here](https://about.gitlab.com/features/gitlab-ci-cd/). From our perspective the most important commands
are `eb use <env name>` and `eb deploy <env name>` which are responsible for new version deployment.

In case when you deploy not-python application the script will be very similar (the best option is to use image python to run deployment):

```
deploy-staging:
    image: python:3.6.3
    stage: deploy
    script:
        - python3 -m pip install awsebcli
        - eb use staging-environment -v
        - eb deploy staging-environment -v
    environment:
      name: staging
      url: http://my.staging.environment
    only:
      - develop
```

The next important key to make it work are credentials. The above runners are from GitLab runners pool so we have to provide somehow AWS secrets.
In order to do that we have to define _Secret variables_ on _GitLab -> Settings -> CI/CD_.

![secrets](/assets/images/2/secrets.png)

As I mentioned before for Multicontainer Docker deployment we have to first prepare docker image in our repo. To do that we could use also Gitlab CI
and example script:

```
dockerize:
    image: docker:latest
    stage: build
    variables:
        DOCKER_DRIVER: overlay2
    services:
        - docker:dind
    before_script:
        - docker info
    script:
        - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN registry.gitlab.com
        - IMAGE_NAME="registry.gitlab.com/mygitlabprofile/mygitlabproject"
        - TAG_XYZ="$(date '+%Y-%m-%d_%H-%M-%S')"
        - TAG_LATEST="latest"
        - VERSION_XYZ="$IMAGE_NAME:$TAG_XYZ"
        - VERSION_LATEST="$IMAGE_NAME:$TAG_LATEST"
        - docker image build -f ./docker/Dockerfile -t $VERSION_LATEST -t $VERSION_XYZ .
        - docker push $IMAGE_NAME
```

Of course you could modify above script in case you expect another tag versioning approach or would like to build images only in specific cases (like
tag, etc.).

In this moment we have configured build process and deployment process but often application requires some environment-specific configuration
(database configuration, etc.). As I mentioned before keep that kind of data on VCS is not a good idea. We could configure environment variables 
on the AWS Elastic Beanstalk console. We will find it on _AWS Services -> Elastic Beanstalk -> Project -> Environment -> Configuration -> Software
Configuration -> Environment Properties_. We could set there all required by our application config properties.

Of course there is more options. We could always change there Scaling, Instances, etc. For single instances it's easy and convenient way but if we have
more environments to mange it would be better to use console.

Ok - so in this moment our application should automatically build and deploy to EB environment(s). The good idea it would be to connect with the 
environment the custom domain instead of default AWS domain name. This operation is very easy even if you don't have domain on the AWS but on the
external domain provider (for domains registered directly on AWS it's event easier - you will find everything on [documentation](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customdomains.html)).

To connect with custom domain you have to:
1. Login to you domain provider.
2. Go to DNS settings.
3. Create `CNAME` record. Unfortunately the main domain (without subdomain) does not work so if you are planning to use your main domain 
the best option here is define CNAME record with `www` on the subdomain field (sometimes this field could have name host or something like that) 
and your AWS DNS domain as a target. 
4. If you using your main domain (`www` as required subdomain) you could also add redirect from you main domain (ie. mydomain.com) 
to created CNAME subdomain (ie. www.mydomain.com). Otherwise this action is not required. 
5. That's all - wait for DNS record refresh.

The Elastic Beanstalk setup for use the EC 2 instances. It could happen that sometimes we will need to go to that EC 2 instances and change something.
Because Docker environments should be stateless (and in multiple instances AWS could setup and terminate instances for us) EB provide the ebextensions.

This feature allows to create command which will be executed on the instances. To do that we have to create `.ebextensions` directory and put there
config files. For example in order to add EC 2 instances user to docker users group and truncate docker logs we have to prepare `docker.config` with
the following content:

```
commands:
  add_ec2_user_to_docker_group:
    command: "sudo usermod -aG docker ec2-user"
    ignoreErrors: true
  add_docker_log_truncate_cronjob:
    command: "crontab -l | grep -q 'eb-docker' || (crontab -l ; echo '* */6 * * * /bin/echo 0 | tee /var/log/eb-docker/containers/*/*.log')| crontab -"
    ignoreErrors: true
```