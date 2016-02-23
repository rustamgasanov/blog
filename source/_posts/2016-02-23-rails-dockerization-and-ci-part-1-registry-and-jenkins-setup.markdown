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
In order to begin, you need `docker` and `docker-compose` to be installed. I won't repeat official guides, you can find detailed instructions here: <a href="https://docs.docker.com/engine/installation/linux/ubuntulinux/" target="_blank">docker</a>, <a href="https://docs.docker.com/compose/install/" target="_blank">docker-compose</a>.

### Setup project skeleton

Create a folder for the docker-compose project in a home directory(`~` or whenever you want), I would call it `myproject-services` and `cd` to it:

```
$ mkdir myproject-services
$ cd myproject-services
```

Then create three folders: `data_nginx`, `data_registry` and `data_jenkins`. Their content would be our shared between running containers and host's filesystems:

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

Here we set `nginx` container to expose ports `80`(HTTP) and `443`(HTTPS), and linking this container with `registry` and `jenkins`(in short words it allows `nginx` container to access them, <a href="https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/" target="_blank">more about docker links</a>), other instructions are pretty self-explanatory. 

As a security measures, we are going to setup [`HTTP Basic Auth`](https://en.wikipedia.org/wiki/Basic_access_authentication) and `SSL` for both services(<a href="https://docs.docker.com/registry/deploying/#running-a-domain-registry" target="_blank">which is required for registry</a>).

### Prepare HTTP Basic Auth

Let's install `apache2-utils` package, which contain `htpasswd` utility:

```
$ sudo apt-get -y install apache2-utils
```

Then `cd` into `data_nginx` and use it to generate passwords for both services(replace `USERNAME` and `PASSWORD` with your pairs):

```
$ cd ~/myproject-services/data_nginx/
$ htpasswd -c registry.password USERNAME
New password: PASSWORD
Re-type new password: PASSWORD
$ htpasswd -c jenkins.password USERNAME
New password: PASSWORD
Re-type new password: PASSWORD
```

### Prepare SSL certificates

I am going to use popular resource <a href="https://letsencrypt.org" target="_black">letsencrypt.org</a> to get free certificates for my domains. Let's clone the repository and generate them:

```
$ cd ~
$ git clone https://github.com/letsencrypt/letsencrypt
$ cd letsencrypt
$ ./letsencrypt-auto certonly --standalone -d registry.myproject.com
$ ./letsencrypt-auto certonly --standalone -d ci.myproject.com
```

It'll generate 4 files(`cert.pem  chain.pem  fullchain.pem  privkey.pem`) for each domain in these folders:
 `/etc/letsencrypt/live/registry.myproject.com/`
 `/etc/letsencrypt/live/ci.myproject.com/`

### Notes

github, chef, auto certs renewal

### Useful links:

<a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04" target="_blank">How to set up private docker registry by Digital Ocean</a>
<a href="https://community.letsencrypt.org/t/howto-easy-cert-generation-and-renewal-with-nginx/3491" target="_blank">Easy cert generation and renewal</a>
