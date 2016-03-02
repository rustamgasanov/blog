---
layout: post
title: "Dockerize Your Project. Part 3: Continuous builds and dockerization with Jenkins."
date: 2016-03-01 22:00:19 +0100
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

In <a href="http://rustamagasanov.com/blog/2016/02/23/dockerize-your-project-part-1-registry-and-jenkins-setup/" target="_blank">the first part</a> of this series we've set up `jenkins` and private `docker registry`. Then, in <a href="http://rustamagasanov.com/blog/2016/02/23/http://rustamagasanov.com/blog/2016/02/24/dockerize-your-project-part-2-building-rails-and-postgres-images-with-docker-compose/" target="_blank">part 2</a> we've created a build configuration and dockerized the `Rails` application manually. Of course we don't want to do this by hand everytime the code in the main branch of the project changes, so the time has come to configure `jenkins` for automated builds and dockerization.

<!-- more -->

In order to build and dockerize `Ruby`(`Rails`) application we would need slighly more tools than official `jenkins` image provides. These tools are:

* ruby
* bundler
* database client
* js runtime
* docker

So first of all we need an image with everything above on top of `jenkins` official image. Let's create it.

Create a new project directory, I would call it `myproject-jenkins`, cd to it:

```
$ mkdir myproject-jenkins
$ cd myproject-jenkins
```

and create a `Dockerfile` with the following content:

```
$ cat Dockerfile
FROM jenkins:1.642.2

USER root
RUN apt-get -qy update
RUN apt-get -qy install dpkg-dev debian-keyring

# ruby2.3 package dependencies
RUN apt-get -qy install dpkg-dev autotools-dev bison chrpath debhelper dh-autoreconf file libffi-dev libgdbm-dev libgmp-dev libncurses5-dev libncursesw5-dev libreadline6-dev libssl-dev libyaml-dev ruby ruby-interpreter  rubygems-integration systemtap-sdt-dev tcl8.5-dev tk8.5-dev

# dbs/runtime dependencies
RUN apt-get -qy install libpq-dev libsqlite3-dev nodejs

# docker
RUN apt-get -qy install apt-transport-https ca-certificates
RUN apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
RUN touch /etc/apt/sources.list.d/docker.list
RUN echo "deb https://apt.dockerproject.org/repo debian-jessie main" > /etc/apt/sources.list.d/docker.list
RUN apt-get -qy update
RUN apt-get -qy install docker-engine
RUN gpasswd -a jenkins docker

ADD build /var/jenkins_home/build
RUN /var/jenkins_home/build/ruby-install.sh
RUN /var/jenkins_home/build/gems-install.sh
RUN /var/jenkins_home/build/change-permissions.sh
RUN rm -Rf /var/jenkins_home/build

USER jenkins
ENTRYPOINT ["/bin/tini", "--", "/usr/local/bin/jenkins.sh"]
```

Packages installation is pretty straightforward, the tricky part starts from the `docker`. The `RUN` steps are just copied <a href="https://docs.docker.com/engine/installation/linux/debian/" target="_blank">from the official page</a> for `Debian Jessie`. But, as you can, see we don't run it as a daemon anywhere. The good explanation why you don't want to run docker inside another docker(dind) container, expecially for CI, you can find in this post by Jérôme Petazzoni: <a href="http://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/" target="_blank">Using Docker-in-Docker for your CI or testing environment? Think twice.</a>. Instead, we would share `docker.sock` and `bin/docker` between the host and `jenkins` container.

After the `docker` setup, I execute 3 simple scripts: `build/ruby-install.sh`, `build/gems-install.sh` and `build/change-permissions.sh`:

```
$ cat build/ruby-install.sh
#!/bin/bash

cd /tmp
wget -O ruby-install-0.6.0.tar.gz https://github.com/postmodern/ruby-install/archive/v0.6.0.tar.gz
tar xvzf ruby-install-0.6.0.tar.gz
cd ruby-install-0.6.0
make install
ruby-install ruby 2.3.0

ln -nfs /opt/rubies/ruby-2.3.0/bin/ruby /usr/bin/ruby
ln -nfs /opt/rubies/ruby-2.3.0/bin/gem /usr/bin/gem
```

```
$ cat build/gems-install.sh
#!/bin/bash

gem install rake bundler --no-rdoc --no-ri
ln -nfs /opt/rubies/ruby-2.3.0/bin/bundle /usr/bin/bundle
ln -nfs /opt/rubies/ruby-2.3.0/bin/bundler /usr/bin/bundler
```

```
$ cat build/change-permissions.sh
#!/bin/bash

chown jenkins:jenkins /opt/rubies -R
```
