---
layout: post
title: "Rails Dockerization and CI. Part 1: Registry and Jenkins setup with docker-compose."
date: 2016-02-23 21:42:24 +0100
comments: true
categories: 
  - ubuntu
  - tutorial
  - example
  - rails
  - ruby
  - docker
  - jenkins
  - registry
  - ci
---

In this series I am going to show and describe the proccess of setting up docker registry, continious integration, dockerization and deployment of your application(s).

The first step is to setup `Docker Registry` as a storage of our images and `Jenkins` for future builds and dockerization.

<!-- more -->

Since these two services are independent of our application(s) I prefer to keep them on separate server. As an OS I will use `Ubuntu 14.04.4`.
In order to begin, you need `docker` and `docker-compose` to be installed. I won't repeat official guides, you can find detailed instructions here: [docker](https://docs.docker.com/engine/installation/linux/ubuntulinux/), [docker-compose](https://docs.docker.com/compose/install/).

Create a folder for the docker-compose project in a home directory(`~` or whenever you want), I would call it `myproject-services` and `cd` to it:

```
$ mkdir myproject-services
$ cd myproject-services
```

Then create three folders: `data_nginx`, `data_registry` and `data_jenkins`. Their content would be our shared between running containers and host OS:

```
$ mkdir data_nginx
$ mkdir data_registry
$ mkdir data_jenkins
```

Using any redactor, create `docker-compose.yml` file with the following instructions:

```
$ cat docker-compose.yml
nginx:
  image: "nginx:1.9"
  ports:
    - 80:80
    - 443:443
  links:
    - registry:registry
    - jenkins:jenkins
  volumes:
    - ./data_nginx/:/etc/nginx/conf.d:ro
registry:
  image: registry:2
  environment:
    REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /registry_data
  volumes:
    - ./data_registry:/registry_data
jenkins:
  image: "jenkins:1.565.3"
  ports:
    - 50000:50000
  volumes:
    - ./data_jenkins:/var/jenkins_home
```

Here we set `nginx` container to expose ports `80`(HTTP) and `443`(HTTPS), and linking this container with `registry` and `jenkins`(in short words it allows `nginx` container to access them, [more about docker links](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/)), other instructions are pretty self-explanatory. 

