---
layout: single
published: true
title: 'Secure web project hosting with docker-compose, nginx, and LetsEncrypt'
---

Thanks to docker, lets-encrypt and nginx we can host many little web projects with full https support on a single cheap VPS.

Perfect for all those side projects that are easy to get going in node, but a pain to wrap with nginx and ssl configurations.

## Goals
 
 * A place to host little nodejs/dotnet-core/ruby projects
 * Ideally the projects can be run locally on developers computer just as easily
 * Expose projects on their own domains or subdomains
 * Provide vhost support with SSL for each project managed for us by letsencrypt
 * Easy to rebuild if changing VPS provider

## Our Solution:

 * A cheap ubuntu VPS running docker and docker-compose
 * A docker-compose network to carry the internal traffic from nginx to our apps
 * A docker instance for each node/ruby/dotnet core endpoint to sit behind nginx
 * A docker instance that runs nginx and auto configures vhosts and renews and obtains letsencrypt certificates
  

## A cheap ubuntu VPS running docker and docker compose

Grab yourself a cheap ubuntu VPS (16.04 LTS preferrably). I use [Binary Lane](https://www.binarylane.com.au/) (because the developers are awesome).

To get docker up and running we follow [this guide.](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-using-the-repository)

Docker installed from package doesn't include docker-compose. Instead you are meant to grab the latest binary from the [docker compose releases](https://github.com/docker/compose/releases) page on github.

Grab one and install it as explained on the releases page.


## A docker-compose network to carry the internal traffic from nginx to our apps

We would like the docker containers to all be able to talk to each other, preferrably through some internal interfaces so that each application can use its preferred port numbers and remains hidden from the public accessible IP of the VPS. 

![container-layout.png]({{site.baseurl}}/assets/images/container-layout.png)

By default docker containers are on the `default` network. To help keep our web accessible containers apart from any other fiddling we might do on the host, docker allows you to define an alternate internal network and give it a name. Docker compose configurations can reference the network by name and expose services on the network with aliases. This allows the nginx container to proxy traffic to the application containers without having the application containers specified in the same docker-compose configuration file.

to create a network called `nginx` we run:

```sh
$ sudo docker network create nginx
```

With our network spun up we can add our first application.

## A docker instance for each node/ruby/dotnet core endpoint to sit behind nginx

We will spin up a simple nodje hellow world application as an example.

Create a folder called `hello-world` and paste the following into `docker-compose.yml`

```yml

version: '3'
services:

  hello-world:
    image: ashleybarrett/node-hello-world
    restart: always
    networks:
      nginx:
        aliases:
          - helloworld

networks:
  nginx:
    external: true
```

This configures a service called `hello-world` from an image I found on [docker-hub](https://hub.docker.com/r/ashleybarrett/node-hello-world/), and runs on the `nginx` network (we have already configured) with the alias `helloworld`

The `networks` section at the bottom tells docker-compose that we have already defined the `nginx` network and that it is external to this file. It will look for, and attach to, the network when we spin up the services. 

Note that we didn't need to expose any ports, any ports opened by the application will be visible on the nginx network.{: .notice}

We can test our compose file by running `sudo docker-compose up -d` in the `hello-world` folder, and we should see it download the image and start the container:

```
$ sudo docker-compose up -d

Pulling hello-world (ashleybarrett/node-hello-world:latest)...
latest: Pulling from ashleybarrett/node-hello-world
357ea8c3d80b: Pull complete
52befadefd24: Pull complete
3c0732d5313c: Pull complete
ceb711c7e301: Pull complete
868b1d0e2aad: Pull complete
3a438db159a5: Pull complete
894fe2afbe58: Pull complete
3caf831c6cd9: Pull complete
f4697cfa8036: Pull complete
235dc8c3c429: Pull complete
Digest: sha256:77a49ad3cf2016dc0b12a1e0687d450e11040cc27438f825e90b230143a775c1
Status: Downloaded newer image for ashleybarrett/node-hello-world:latest
Creating helloworld_hello-world_1

$
```

Since we didn't expose any of the ports, we need to find its IP address on the `nginx` network. There is handy docker command to show all containers on a network

```sh
$ docker network inspect nginx
---snip---
       "Containers": {
            "2472ae13d79b672675324da51932cd514166c15de598ef4bd317e1cf23e03f06": {
                "Name": "helloworld_hello-world_1",
                "EndpointID": "02570dabe70651b8889c4c8bac14bae2057b9144f1bbebfdbe8ee45cb328573e",
                "MacAddress": "02:42:ac:12:00:05",
                "IPv4Address": "172.18.0.5/16",
                "IPv6Address": ""
            }
---snip---
$
```

We find our container ip and hit port 8080 with curl

```sh
$ curl 172.18.0.5:8080
Hello world
$
```

Our container is exposed on the nginx network ready for us to put an nginx container in front.


## A docker instance that runs nginx and auto configures vhosts and renews and obtains letsencrypt certificates

To proxy incoming web requests to the relevant docker containers and to provide SSL termination for those requests, we need an instance of nginx configured with each docker container as a vhost and certificates for each domain. We will use the https-portal docker container that is introduced [in this blog post](https://steveltn.me/a-brief-introduction-to-https-portal-80bb3286eebc)
and is hosted on github [here](https://github.com/SteveLTN/https-portal)

Create a folder next to `hello-world` called `https-portal` and create a docker-compose.yml with the contents:

```yml

version: '3'
services:

  https-portal:
    image: steveltn/https-portal:1
    ports:
      - '80:80'
      - '443:443'
    restart: always
    networks:
      - nginx
    environment:
      - "DOMAINS=yourdomain.com -> http://helloworld:8080"
#      - "PRODUCTION=true"


networks:
  nginx:
    external: true

```

This defines the `https-portal` service based on the image mentioned above. It exposes 80 and 440. The environment section allows us to specify which domains map to which docker containers and on what port.

The docker has letsencrypt support built in and will by default get you a staging certificate (which isn't secure) to ensure your scripts are configured correctly without hitting lets-encrypt's usage limits. When you have it all happy, uncomment the `PRODUCTION=true` line

Spin it up with 

```sh
$ sudo docker-compose up -d
```

and you should be able to hit `http://yourdomain.com` and see `Hello World`. You should also be able to hit `https://yourdomain.com` and see that you have a secure site. The certificates are automatically renewed by the nginx docker container and are free.

To add another applications, just add another docker container on the nginx network, and add the domain to the domains environment variable config for https-portal 

```yml
- "DOMAINS=app1.yourdomain.com -> http://app1:8080, app2.yourdomain.com -> http://app2:8080"
```
