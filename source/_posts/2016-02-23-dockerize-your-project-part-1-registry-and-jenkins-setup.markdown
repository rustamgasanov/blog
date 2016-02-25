---
layout: post
title: "Dockerize your project. Part 1: Docker Registry and Jenkins setup with docker-compose."
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

In this series of articles I am going to show and describe how to dockerize your application(s) and services, and how to setup continious integration and dockerization with Jenkins.

The first step is to setup `Docker Registry` as a storage of our images and `Jenkins` for future builds and dockerization.

<!-- more -->

Since these two services are independent of our application(s) I prefer to keep them on separate server. As an OS I will use `Ubuntu 14.04.4`.
In order to begin, you need `docker` and `docker-compose` to be installed. I won't repeat official guides, you can find detailed instructions here: <a href="https://docs.docker.com/engine/installation/linux/ubuntulinux/" target="_blank">docker</a>, <a href="https://docs.docker.com/compose/install/" target="_blank">docker-compose</a>.

### Project skeleton

After installation is done, create a folder for the `docker-compose` project in the home directory(`~` or whenever you want), I would call it `myproject-services`, then `cd` to it:

```
$ mkdir myproject-services
$ cd myproject-services
```

Create three folders: `data_nginx`, `data_registry` and `data_jenkins`. Their content would be shared between running containers and host's filesystems:

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

Here we set `nginx` container to expose ports `80`(HTTP) and `443`(HTTPS), and linking it with `registry` and `jenkins`(in short words it allows `nginx` container to access them, <a href="https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/" target="_blank">more about docker links</a>), other instructions are pretty self-explanatory. 

As a security measures, we are going to setup [`HTTP Basic Auth`](https://en.wikipedia.org/wiki/Basic_access_authentication) and `SSL` for both services(<a href="https://docs.docker.com/registry/deploying/#running-a-domain-registry" target="_blank">it's required for registry</a>).

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

It'll generate 4 files(`cert.pem  chain.pem  fullchain.pem  privkey.pem`) for each domain in the following folders:

* `/etc/letsencrypt/live/registry.myproject.com/`
* `/etc/letsencrypt/live/ci.myproject.com/`

We would need `fullchain.pem` and `privkey.pem` for `nginx` config, so let's create corresponding directories in `data_nginx` and copy the certs we need:

```
$ cd ~/myproject-services/data_nginx/
$ mkdir jenkins_ssl_keys
$ mkdir registry_ssl_keys
$ sudo cp /etc/letsencrypt/live/registry.myproject.com/fullchain.pem /etc/letsencrypt/live/registry.myproject.com/privkey.pem ~/myproject-services/data_nginx/registry_ssl_keys/
$ sudo cp /etc/letsencrypt/live/ci.myproject.com/fullchain.pem /etc/letsencrypt/live/ci.myproject.com/privkey.pem ~/myproject-services/data_nginx/jenkins_ssl_keys/
```

These files are owned by `root` right now, so we need to change the owner to our user:

```
$ sudo chown $USER:$USER ~/myproject-services/data_nginx/registry_ssl_keys/*
$ sudo chown $USER:$USER ~/myproject-services/data_nginx/jenkins_ssl_keys/*
```

### Nginx configs

Finally we need to prepare `nginx` configs for both services. Create `registry.conf` and `jenkins.conf` files inside `data_nginx` folder with the following content:

```
$ cat ~/myproject-services/data_nginx/registry.conf
upstream docker-registry {
  server registry:5000;
}

server {
  listen 443 ssl;
  server_name registry.myproject.com;

  ssl_certificate         /etc/nginx/conf.d/registry_ssl_keys/fullchain.pem;
  ssl_certificate_key     /etc/nginx/conf.d/registry_ssl_keys/privkey.pem;
  ssl_trusted_certificate /etc/nginx/conf.d/registry_ssl_keys/fullchain.pem;
  ssl_session_timeout 1d;
  ssl_session_cache shared:SSL:50m;

  # What Mozilla calls "Intermediate configuration"
  # Copied from https://mozilla.github.io/server-side-tls/ssl-config-generator/
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:ECDHE-RSA-DES-CBC3-SHA:ECDHE-ECDSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
  ssl_prefer_server_ciphers on;

  # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
  add_header Strict-Transport-Security max-age=15768000;

  # OCSP Stapling
  # fetch OCSP records from URL in ssl_certificate and cache them
  ssl_stapling on;
  ssl_stapling_verify on;

  # disable any limits to avoid HTTP 413 for large image uploads
  client_max_body_size 0;

  # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
  chunked_transfer_encoding on;

  location /v2/ {
    # Do not allow connections from docker 1.5 and earlier
    # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents
    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
      return 404;
    }

    # To add basic authentication to v2 use auth_basic setting plus add_header
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/conf.d/registry.password;
    add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;

    proxy_pass                          http://docker-registry;
    proxy_set_header  Host              $http_host;   # required for docker client's sake
    proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
    proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header  X-Forwarded-Proto $scheme;
    proxy_read_timeout                  900;
  }
}
```

```
$ cat ~/myproject-services/data_nginx/jenkins.conf
upstream docker-jenkins {
  server jenkins:8080;
}

server {
  listen 443 ssl;
  server_name ci.myproject.com;

  ssl_certificate         /etc/nginx/conf.d/jenkins_ssl_keys/fullchain.pem;
  ssl_certificate_key     /etc/nginx/conf.d/jenkins_ssl_keys/privkey.pem;
  ssl_trusted_certificate /etc/nginx/conf.d/jenkins_ssl_keys/fullchain.pem;
  ssl_session_timeout 1d;
  ssl_session_cache shared:SSL:50m;

  # What Mozilla calls "Intermediate configuration"
  # Copied from https://mozilla.github.io/server-side-tls/ssl-config-generator/
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:ECDHE-RSA-DES-CBC3-SHA:ECDHE-ECDSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
  ssl_prefer_server_ciphers on;

  # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
  add_header Strict-Transport-Security max-age=15768000;

  # OCSP Stapling
  # fetch OCSP records from URL in ssl_certificate and cache them
  ssl_stapling on;
  ssl_stapling_verify on;

  # disable any limits to avoid HTTP 413 for large image uploads
  client_max_body_size 0;

  # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
  chunked_transfer_encoding on;

  location / {
    # Do not allow connections from docker 1.5 and earlier
    # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents
    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
      return 404;
    }

    # To add basic authentication use auth_basic setting plus add_header
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/conf.d/jenkins.password;

    proxy_pass                          http://docker-jenkins;
    proxy_set_header  Host              $http_host;   # required for docker client's sake
    proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
    proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header  X-Forwarded-Proto $scheme;
    proxy_read_timeout                  900;
  }
}
```

The config is mostly taken from <a href="https://community.letsencrypt.org/t/nginx-configuration-sample/2173" target="_blank">letsencrypt.org nginx sample</a>. In the `upstream` stanza we're specifying linked containers and ports on which these services are running.

### Run services

By this time your `~/myproject-services/` directory should have the following structure:

{% img /images/dockerization_services_structure.png %}

Finally we can start all services with:

```
$ cd ~/myproject-services/
$ docker-compose up -d
```

`-d` flag tells `docker-compose` to run as a daemon. Now, two useful commands to ensure everything is ok are `ps` and `logs`:

```
$ docker-compose ps
           Name                         Command               State                    Ports
--------------------------------------------------------------------------------------------------------------
yourprojectservices_jenkins_1    /usr/local/bin/jenkins.sh        Up      0.0.0.0:50000->50000/tcp, 8080/tcp
yourprojectservices_nginx_1      nginx -g daemon off;             Up      0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
yourprojectservices_registry_1   /bin/registry /etc/docker/ ...   Up      5000/tcp
$ docker-compose logs
...
```

And of course both services are now should be live and available at `https://registry.myproject.com` and `https://ci.myproject.com`. To stop them, type:

```
$ docker-compose down
```

### Final notes

What you now probably want to do and what I left behind the scene:

* add this project to `git`, ignoring artifacts(certs), but keeping empty folders(`data_jenkins`, `data_registry`)
* add cron job for automated ssl certs renewal(letsencrypt generates certs valid for 3 months)
* add monitoring tools
* add backup system
* add any kind of automatization for the host initial setup(`docker`, `docker-compose`, cron, monitoring, etc) using `chef`, `puppet` or anything else

### Useful links:

<a href="https://docs.docker.com/compose/reference/" target="_blank">Docker compose reference</a><br>
<a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04" target="_blank">How to set up private docker registry by Digital Ocean</a><br>
<a href="https://community.letsencrypt.org/t/howto-easy-cert-generation-and-renewal-with-nginx/3491" target="_blank">Easy cert generation and renewal</a>
