---
layout: post
title: "Dockerize Your Project. Part 2: Building Rails and Postgres images with docker-compose."
date: 2016-02-24 20:46:41 +0100
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

Previous parts:
<a href="http://rustamagasanov.com/blog/2016/02/23/dockerize-your-project-part-1-registry-and-jenkins-setup/" target="_blank">Part 1: Docker Registry and Jenkins setup with docker-compose.</a>

In this tutorial I'll show how to create a `docker` image with your Rails application, push it to the private `Docker Registry`(that we setup in Part 1), and then how to run it with `postgres` database image, wrapping everything in `docker-compose.yml`.

<!-- more -->

### Base image creation

Usually nowadays you would have more than one Ruby-application(micro-service) in your apps stack, so I'll begin with creation of the base image for them. Let's call it `myproject-baseimage`. Create a new folder and `cd` to it:

```
$ mkdir myproject-baseimage
$ cd myproject-baseimage
```

I picked <a href="https://hub.docker.com/r/phusion/baseimage/" target="_blank">phusion/baseimage:0.9.18</a> as a base for our new image because it's ligthweight and prepared to run as a `docker` container. We are going to add `ruby 2.3` and `bundler` to it. 

Open your favorite redactor and create the `Dockerfile` with the following content:

```
$ cat Dockerfile
FROM phusion/baseimage:0.9.18

RUN apt-get -qy update && apt-get -qy install unzip git-core build-essential libsqlite3-dev libpq-dev

ADD build /tmp/build
RUN /tmp/build/ruby-install.sh
RUN /tmp/build/gems-install.sh
RUN rm -Rf /tmp/build

ENTRYPOINT ["/sbin/my_init"]
CMD ["/bin/bash"]
```

In this file we set up essentials libs, then execute two scripts: `build/ruby-install.sh`, `build/gems-install.sh` - let's create them:

```
$ mkdir build
$ vim build/ruby-install.sh
#!/bin/bash

apt-add-repository ppa:brightbox/ruby-ng
apt-get -qy update
apt-get install -qy ruby2.3 ruby2.3-dev
$ vim build/gems-install.sh
#!/bin/bash

gem install rake bundler --no-rdoc --no-ri
```

And make them executable:

```
$ chmod 755 build/ruby-install.sh
$ chmod 755 build/gems-install.sh
```

Now, when everything is set, we can start the `myproject/baseimage` build:

```
$ docker build -t myproject/baseimage .
```

And push it to out private registry:

```
$ docker tag myproject/baseimage registry.myproject.com/myproject/baseimage
$ docker push registry.myproject.com/myproject/baseimage
```

After the execution is complete, you can check the catalog of the registry via `https://registry.myproject.com/v2/_catalog`, it'll output:

```
{"repositories":["myproject/baseimage"]}
```

Also, you can list the tags for this image via `https://registry.myproject.com/v2/myproject/baseimage/tags/list`

```
{"name":"myproject/baseimage","tags":["latest"]}
```

Since during the build we didn't tag it with any specific version, by default `docker` set the tag `latest`. Now we can use this image to dockerize our Ruby/Rails application(s).

Create new project, I would call it `myproject-buildconfigs`, which will contain the `docker` build instructions for all our applications(and will be used by Jenkins in the future). Then create a directory with a build for our first app inside:

```
$ mkdir myproject-buildconfigs
$ mkdir myproject-buildconfigs/myproject-app1
```


