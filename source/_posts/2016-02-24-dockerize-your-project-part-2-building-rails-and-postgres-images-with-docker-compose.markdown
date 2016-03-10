---
layout: post
title: "Dockerize Your Project. Part 2: Building Rails and Postgres images."
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

Previous part:
<a href="http://rustamagasanov.com/blog/2016/02/23/dockerize-your-project-part-1-registry-and-jenkins-setup/" target="_blank">Part 1: Docker Registry and Jenkins setup with docker-compose.</a>

In this tutorial I'll show how to create a docker image with your Rails application, push it to the private docker registry(that we set up in Part 1), and then how to run it with postgres database image, wrapping everything in `docker-compose.yml`.

<!-- more -->

### Base image creation

Usually nowadays you would have more than one Ruby-application in your apps stack, so I'll begin with a creation of the base image for them. Let's call it `myproject-baseimage`. Create a new folder and `cd` to it:

```
$ mkdir myproject-baseimage
$ cd myproject-baseimage
```

I picked <a href="https://hub.docker.com/r/phusion/baseimage/" target="_blank">phusion/baseimage:0.9.18</a> as a base for our new image because it's ligthweight and Ubuntu inside is prepared to run as a docker container. We are going to add `ruby 2.3`, `bundler` and js-runtime(`nodejs`) to it. 

Open your favorite text editor and create the `Dockerfile` with the following content:

```
$ cat Dockerfile
FROM phusion/baseimage:0.9.18

RUN apt-get -qy update && apt-get -qy install unzip git-core build-essential libsqlite3-dev libpq-dev nodejs

ADD build /tmp/build
RUN /tmp/build/ruby-install.sh
RUN /tmp/build/gems-install.sh
RUN rm -Rf /tmp/build

ENTRYPOINT ["/sbin/my_init"]
CMD ["/bin/bash"]
```

In this file we set up essential libs, then execute two scripts: `build/ruby-install.sh`, `build/gems-install.sh` - let's create them:

```
$ mkdir build
```

```
$ vim build/ruby-install.sh
#!/bin/bash

apt-add-repository ppa:brightbox/ruby-ng
apt-get -qy update
apt-get install -qy ruby2.3 ruby2.3-dev
```

```
$ vim build/gems-install.sh
#!/bin/bash

gem install rake bundler --no-rdoc --no-ri
```

And make them executable:

```
$ chmod 755 build/ruby-install.sh
$ chmod 755 build/gems-install.sh
```

Final structure:

{% img /images/dockerization_baseimage_structure.png %}

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

> {"repositories":["myproject/baseimage"]}

Also, you can list the tags for this image via `https://registry.myproject.com/v2/myproject/baseimage/tags/list`

> {"name":"myproject/baseimage","tags":["latest"]}

Since during the build we didn't tag it with any specific version, by default docker sets the version `latest`. Now we can use this image to dockerize our Ruby apps.

### Rails application dockerization

Create a new project, I would call it `myproject-buildconfigs`, which will contain the docker build instructions for the apps(and will be used by Jenkins in the future). Then create a directory for our first Rails app inside:

```
$ mkdir myproject-buildconfigs
$ mkdir myproject-buildconfigs/myproject-app1
$ cd myproject-buildconfigs/myproject-app1
```

Initialize the `Dockerfile` inside with the following content:

```
$ cat Dockerfile
FROM registry.myproject.com/myproject/baseimage:latest

ADD app /app
ADD scripts/ /
ADD etc /etc

RUN chmod u+x /start /setup
RUN /sbin/my_init /setup

ENV RAILS_ENV production

EXPOSE 3001

ENTRYPOINT ["/sbin/my_init"]
CMD ["/start"]
```

We would use three directories: `app`, `scripts` and `etc`.

* `app` would be a space for the application
* `scripts/setup` would perform `bundle install` and `rake assets:precompile`
* `scripts/start` would create and migrate the database, then start the app in production environment
* `etc/myproject-app1/config-unicorn.rb` will contain a `unicorn` config

Let's add these files:

```
$ mkdir app scripts etc etc/myproject-app1
```

```
$ vim scripts/setup
#!/bin/bash

cd /app && \
    bundle install --deployment --without development test && \
    SECRET_KEY_BASE=abc DATABASE_URL=sqlite3:tmp/tmp.db RAILS_ENV=production bundle exec rake assets:precompile
```

```
$ vim scrips/start
#!/bin/bash

cd /app
bundle exec rake db:create &&\
bundle exec rake db:migrate &&\
bundle exec unicorn -N -p 3001 -c /etc/myproject-app1/config_unicorn.rb
```

```
$ vim etc/myproject-app1/config-unicorn.rb
worker_processes Integer(ENV['unicorn_worker_count'] || 1)
timeout Integer(ENV['unicorn_worker_timeout'] || 30)

stdout_logger = ::Logger.new($stdout)
$stdout.sync = true
stdout_logger.level = ::Logger::INFO
stdout_logger.formatter = lambda do |severity, _time, _prog, message|
  "#{severity.to_s[0]}, #{message.gsub("\n", '↲')}\n"
end
logger stdout_logger

# preload_app true

before_fork do |_server, _worker|
  Signal.trap 'TERM' do
    puts 'Unicorn master intercepting TERM and sending myself QUIT instead'
    Process.kill 'QUIT', Process.pid
  end

  ActiveRecord::Base.connection.disconnect! if defined?(ActiveRecord::Base)
end

after_fork do |_server, _worker|
  Signal.trap 'TERM' do
    puts 'Unicorn worker intercepting TERM and doing nothing. Wait for master to send QUIT'
  end

  ActiveRecord::Base.establish_connection if defined?(ActiveRecord::Base)
end
```

Final structure:

{% img /images/dockerization_app_build_structure.png %}


In the next part I'll describe how to use this build configuration with Jenkins to automatically dockerize new releases, for now let's try to build it manually and understand the process. Clone your project into app directory, start the build and push it to the registry:

```
$ git clone *repo* -b *branch* --single-branch app
$ docker build -t myproject-apps/myproject-app1 .
$ docker tag myproject-apps/myproject-app1 registry.myproject.com/myproject-apps/myproject-app1
$ docker push registry.myproject.com/myproject-apps/myproject-app1
```

And let's check the catalog(`https://registry.myproject.com/v2/_catalog`) of the registry afterwards:

> {"repositories":["myproject/baseimage","myproject-apps/myproject-app1"]}

Great, we've just created the first dockerized release of our Rails application! Now the final step is to make sure the image works properly. I prefer to use `postrges` as a database, so we would need `postgres` image as well.

### Getting everything up and running

Since we need to start more than one container, let's create `docker-compose` project. I'll call it `myproject-testenv`. There is <a href="https://hub.docker.com/_/postgres/" target="_blank">an official postgres image</a> in docker hub so let's pick it. We would also need a directory to store postgres data:

```
$ mkdir myproject-testenv
$ cd myproject-testenv
$ mkdir data_pg
```

Next, initialize `docker-compose.yml` file with the following content:

```
$ cat docker-compose.yml
myproject-app1:
  image: registry.myproject.com/myproject-apps/myproject-app1
  ports:
    - 3001:3001
  links:
    - postgresql:postgresql
  environment:
    SECRET_KEY_BASE: "MyProjectSecretKeyBase"
    DATABASE_URL: "postgresql://pguser:pgpass@postgresql/pgtable"
postgresql:
  image: postgres:9.5
  environment:
    POSTGRES_USER: "pguser"
    POSTGRES_PASSWORD: "pgpass"
    POSTGRES_DB: "pgtable"
  volumes:
    - ./data_pg:/var/lib/postgresql/data
  ports:
    - 5432:5432
```

Our unicorn starts the app on the port `3001`, so we're exposing it. `app1` will be able to discover postgres container with a help of linking. As environment variables we set required for Rails `SECRET_KEY_BASE` and database location with `DATABASE_URL`. For postgres container, we're setting username/password pair, database name and specifying the `data_pg` directory on the host machine as a volume. In the end we're exposing port 5432, so database would be available on the host.

Now we can start containers with `docker-compose up -d`. Let's check if everything is running:

```
$ docker-compose ps
                 Name                               Command               State            Ports
---------------------------------------------------------------------------------------------------------
myprojecttestenv_myproject-app1_1          /sbin/my_init /start             Up       0.0.0.0:3001->3001/tcp
myprojecttestenv_postgresql_1              /docker-entrypoint.sh postgres   Up       0.0.0.0:5432->5432/tcp
$ docker-compose logs
...
$ curl localhost:3001
...
```

### Final notes

I prefer to develop inside a `vagrant` box and it could be convinient to create a configured box with docker installed and ports on which apps are running exposed to the host. Then you can share this box and `myproject-testenv` with other developers. Say, you want to make changes in `myproject-app1`, the development flow should look like this:

1. Start all the containers inside the box with `docker-compose up -d` 
2. Shut `myproject-app1` with `docker-compose stop myproject-app1`
3. Link the `myproject-app1` directory on your host machine to vagrant box
4. Start `myproject-app1` inside the box with `bundle exec rails s -p 3001` or `bundle exec unicorn -p 3001`

Now apps stack is running inside `vagrant` box and you should see all the changes you make on your host on the exposed port `3001`. The advantages of this approach are:

* consistent and reproducible development environment within the team
* you don't need anything except a `git` and code redactor on your host machine

You can find basic Vagrant box config running Ubuntu 14.04 with Ruby and Docker <a href="https://github.com/rustamagasanov/ruby-devenv" target="_blank">in my repository</a>.
