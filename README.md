[TOC]

# Docker-compose + Traefik + Nextcloud

Another guide to set up a nextcloud with docker-compose and traefik as a reverse-proxy. 

## Why?

Because the guides I found during my installation process were lacking in explanation or outdated. Or both.

## What's the goal?

We want a self-hosted server, using containered services and a modularized docking system, where containers can easily be added on-the-fly without much maintenance. We also want Letsencrypt certs for all our subdomains.

## How do we do it?

Well, obviously with Docker, Traefik and Nextcloud ;-). With Docker we will get containers, Docker-compose is a service which helps us to configure many Docker instances at once. Traefik is a reverse-proxy which will handle the routing to our containers and additionally will manage Letsencrypt certs for us. Nextcloud is a service which manages your files, calenders, contacts etc. I chose Nextcloud for my needs, but it can be run along or be replaced by your containers.

## Okay, lets go!

This assumes that you already have [Docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-using-the-repository) and [Docker-compose](https://docs.docker.com/compose/install/) installed.

In this repository there are currently 2 directories. One contains the docker-compose contents for nextcloud, and one for traefik. Each one contains a `docker-compose.yaml`, which defines how docker-compose should handle the containers. 

I will not explain what every option in the files does. I just want to warn you of some pitfalls and maybe unintuitive configurations.

Let's look at traefik first:

### Traefik

#### docker-compose.yaml

```yaml
services:
  traefik:
    […]
    networks:
      - web
```

This has to be in every service which should be exposed to the outside world.



```yaml
services:
  traefik:
    […]
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.frontend.rule=Host:traefik.example.com"
      - "traefik.port=8080"
      - "traefik.frontend.auth.basic=admin:$$apr1$$PhaXSMTW$$k/P5oLpvRcTIG4bnbn/g9/"
```

These labels must be declared in every container which should be exposed to the outside world. We want to show the traefik interface at traefik.example.com, therefore these labels have to be here.

In basic auth, it is **important** that you escape each $ with another $!



```yaml
[…]
networks:
  web:
    external: true
```

Declares that the network `web` is exposed.

### Nextcloud

#### docker-compose.yaml

```yaml
services:
  db:
    […]
    networks:
      - default

app:
    […]
    networks:
      - web
      - default
```

This is really important. You need to define a network for you DB and your app so they can see each other. In this case I named the network "default".



```yaml
    […]
    labels:
      - "traefik.backend=nextcloud"
      - "traefik.docker.network=web"
      - "traefik.enable=true"
      - "traefik.frontend.rule=Host:nc.example.com"
      - "traefik.port=80"
```

We need to set the traefik.port to 80. Most guides define a port mapping from 8080 (on the host) to 80 (on the container) so they can access the service on localhost:8080. We need to choose port 80 though, as traefik directly communicates with the service within the container, and nextcloud internally exposes port 80.

#### Adding Redis

Redis is a cache server, which will improve your web interface experience.

Just add Redis to your docker-compose:

```yaml
  redis:
    image: redis
    container_name: redis
    volumes:
      - /docker/nextcloud/redis:/data
    networks:
      - internal
```

We still need to point nextcloud to redis. This is done in `config/config.php` (in the container). We will configure the config from outside and load it in when we create the container:

```bash
# On the host system
mkdir -p nextcloud/config
vim nextcloud/config/config.php
# Add your config here. You can also just copy your current config from the container.
# Then add redis at the end
<?php
[…]
'redis' => array(
      'host' => 'redis',
      'port' => 6379,
       ),
);
```

Then adjust your docker-compose to point to your config:

```yaml
services:
  app:
    volumes:
      - nextcloud:/var/www/html # Pulls from /var/lib/docker/volumes/nextcloud_nextcloud/_data/
      - ./nextcloud/config:/var/www/html/config # Pulls from local dir
      - /mnt/someHDD/nextcloud:/mnt/hdd # Pulls from root
```

You might need to adjust the owner of the config dir in your container, or nextcloud can not write to it.

## Troubleshooting

If you want to troubleshoot a container, start it with `docker-compose up` (without the `-d` option).

If your subdomain.example.com throws an HTTPS certificate error due to it being self-signed, it could be because Traefik tries to get the certificate but is not fast enough. Sometimes it helps to just wait a few minutes.
