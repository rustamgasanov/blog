---
layout: post
title: "Dockerize Your Project. Part 3: Rails app continuous builds and dockerization with Jenkins."
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

In <a href="http://rustamagasanov.com/blog/2016/02/23/dockerize-your-project-part-1-registry-and-jenkins-setup/" target="_blank">the first part</a> of this series we've set up Jenkins and private docker registry. Then, in <a href="http://rustamagasanov.com/blog/2016/02/23/http://rustamagasanov.com/blog/2016/02/24/dockerize-your-project-part-2-building-rails-and-postgres-images-with-docker-compose/" target="_blank">part 2</a> we've created a build configuration and dockerized the Rails application manually. Of course we don't want to do this by hand every time the code changes, so the time has come to configure Jenkins for automated builds and dockerization.

<!-- more -->

In order to build and dockerize Rails application we would need slighly more tools than official Jenkins image provides. These tools are:

* ruby
* bundler
* database client
* js runtime
* docker

So, first of all, we need an image with everything above on top of the Jenkins official image. Let's create it.

### Building Jenkins image with Ruby inside

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

# db libs/js runtime
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

Packages installation is pretty straightforward, the tricky part starts from the docker setup. The `RUN` steps are just copied <a href="https://docs.docker.com/engine/installation/linux/debian/" target="_blank">from the official page</a> for Debian Jessie. But, as you can see, we don't run it as a daemon anywhere. The good explanation why you don't want to run docker inside another docker(dind) container, especially for CI, you can find in this post by Jérôme Petazzoni: <a href="http://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/" target="_blank">Using Docker-in-Docker for your CI or testing environment? Think twice.</a> Instead, we would share `docker.sock` and `bin/docker` between the host and jenkins container.

After the docker setup, I execute 3 simple scripts:

* `build/ruby-install.sh`
* `build/gems-install.sh`
* `build/change-permissions.sh`

Here they are:

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

* `ruby-install.sh` installs `ruby2.3` with a help of `ruby-install` tool
* `gems-install.sh` installs `bundler`
* `change-permissions.sh` sets `jenkins` user as owner of the `/opt/rubies`(since `jenkins` user performs builds)

Final structure of the `myproject-jenkins`:

{% img /images/dockerization_jenkins_structure.png %}

Now we can build this image and push it to the registry:

```
$ docker build -t myproject/myproject-jenkins .
$ docker tag myproject/myproject-jenkins registry.myproject.com/myproject/myproject-jenkins
$ docker push registry.myproject.com/myproject/myproject-jenkins
```

### Launch

Replace standard `jenkins` in your `myproject-services` project, that we created in <a href="http://rustamagasanov.com/blog/2016/02/23/dockerize-your-project-part-1-registry-and-jenkins-setup/" target="_blank">part 1</a>. The Rails application also uses postgres, so we need it as well to run specs. Eventually `docker-compose.yml` file should look like this:

``` yaml
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
  image: registry.myproject.com/myproject/myproject-jenkins:latest
  ports:
    - 50000:50000
  links:
    - postgresql:postgresql
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - /usr/bin/docker:/bin/docker
    - ./data_jenkins:/var/jenkins_home
postgresql:
  image: postgres:9.5
  environment:
    POSTGRES_USER: "pguser"
    POSTGRES_PASSWORD: "pgpass"
    POSTGRES_DB: "myproject_test"
  ports:
    - 5432:5432
```

Execute `docker-compose up -d` to start this stack.

### Configuring

#### Plugins setup

I usually use 2 plugins with Jenkins:

* <a href="https://wiki.jenkins-ci.org/display/JENKINS/Git+Plugin" target="_blank">Git Plugin</a> to fetch git repositories
* <a href="https://wiki.jenkins-ci.org/display/JENKINS/EnvInject+Plugin" target="_blank">EnvInject Plugin</a> to use custom environment variables in build instructions

You can either download them manually and put into `data_jenkins/plugins` folder(don't forget about dependencies) or install it from Jenkins web-interface: `https://ci.myproject.com/pluginManager/available`

We would have 2 builds for the app:

1. First ensures that specs are passing and production requirements are met
2. Second creates an image ready for production and puts it into registry

#### Application build

Add a new build, I would call it `myproject-app1`.

* Check `Prepare an environment to run` → `Keep Jenkins Environment Variables` and `Keep Jenkins Build Variables`
* In `Source Code Management` check `Git`, add your repository url and credentials(you can generate ssh-key using `ssh-keygen -t rsa -C "jenkins"` and put it in `data_jenkins/.ssh` folder)<br>
* In `Build Triggers` check `Poll SCM`, every 5 minutes: `H/5 * * * *`<br>
* In `Build Environment` section check `Inject environment variables to the build process` and in `Properties Content` put `DATABASE_URL=postgresql://pguser:pgpass@postgresql/myproject_test`<br>
* In `Build` section add `Execute Shell` with a following content:

```
bundle install --binstubs
bin/rake spec
```

* And when build is successful we want to create an image with it, so in `Post-build Actions` add `Build other projects` → `dockerize myproject-app1` (check `Trigger only if build is stable`)

#### Image creation build

Add downstream build for `myproject-app1`, I named it `dockerize myproject-app1`.

* Check `Prepare an environment to run` → `Keep Jenkins Environment Variables` and `Keep Jenkins Build Variables`. In `Properties Content` put:

```
APP_NAME=myproject-apps/myproject-app1
APP_BUILD_CONFIG_DIR=myproject-app1
APP_BRANCH=master
```

* In `Source Code Management` select `Git`, and add a path to the repository with **build configuration** we created in <a href="http://rustamagasanov.com/blog/2016/02/24/dockerize-your-project-part-2-building-rails-and-postgres-images-with-docker-compose/" targe="_blank">part 2</a> and credentials
* In `Build` section add `Execute Shell` with a following content:

```
cd $APP_BUILD_CONFIG_DIR
rm -rf app
git clone git@gitlab.myproject.com:$APP_NAME.git -b $APP_BRANCH --single-branch app

cd app
APP_REVISION=$(git rev-parse HEAD)

cd ..

docker build -t $APP_NAME:$APP_REVISION .
docker tag $APP_NAME:$APP_REVISION $APP_NAME:latest
docker tag $APP_NAME:$APP_REVISION registry.myproject.com/$APP_NAME
docker tag $APP_NAME:$APP_REVISION registry.myproject.com/$APP_NAME:$APP_REVISION
docker push registry.myproject.com/$APP_NAME
docker push registry.myproject.com/$APP_NAME:$APP_REVISION
docker rmi registry.myproject.com/$APP_NAME:$APP_REVISION
```

#### One more step

Since our private registry is protected with HTTP Basic Auth, we need to login Jenkins. To do this, enter the container with

```
$ docker ps 
$ docker exec -it *container_id* bash
```

And inside the container, type

```
$ docker login -e "jenkins@myproject.com" -u "username" -p "password" registry.myproject.com
```

It will create `config.json` file in `data_jenkins/.docker/` directory with authentication details.

That's it. Now you can leave the container and start the first build!
