---
layout: single
published: false
title: Reusable secure web project hosting with docker-compose
---

Thanks to docker-compose, lets-encrypt and VPS providers we can cheaply host many little web projects with full https support including auto renewing certificates

## Goals
 
 * A place to host little nodejs/dotnet-core/ruby projects
 * Ideally the projects can be run locally on developers computer just as easily
 * Expose projects on their own domains or subdomains
 * Provide vhost support with SSL for each project managed
 * Easy to rebuild if changing VPS provider

## Our Solution:

 * A cheap ubuntu VPS running docker and docker-compose
 * A docker-compose network to carry the internal traffic from nginx to our apps
 * A docker instance for each node/ruby/dotnet core endpoint to sit behind nginx
 * A docker instance that runs nginx and auto configures vhosts and renews and obtains letsencrypt certificates
  

### A cheap ubuntu VPS running docker and docker compose

Grab yourself a cheap ubuntu VPS (16.04 LTS preferrably). I use https://www.binarylane.com.au/ because the developers are awesome.

To get docker up and running you follow this guide [this guide.](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-using-the-repository)

Docker installed from package doesn't include docker-compose. Instead you are meant to grab the latest binary from the [docker compose releases](https://github.com/docker/compose/releases) page on github.
Grab one and install it as explained on the releases page.


### A docker-compose network to carry the internal traffic from nginx to our apps

We would like the docker containers to all be able to talk to each other, preferrably through some internal interfaces so that each application can use its preferred port numbers and remains hidden from the public accessible IP of the VPS. 

![Container Layout]({{site.baseurl}}/assets/images/container-layout.svg)




### A docker instance that runs nginx and auto configures vhosts and renews and obtains letsencrypt certificates

To proxy incoming web requests to the relevant docker containers and to provide SSL termination for those requests, we need an instance of nginx configured with each docker container as a vhost and certificates for each domain. In the interests of not reinventing the wheel, you can just use the https-portal docker container that is introduced [in this blog post](https://steveltn.me/a-brief-introduction-to-https-portal-80bb3286eebc)
and is hosted on github [here](https://github.com/SteveLTN/https-portal)









Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
