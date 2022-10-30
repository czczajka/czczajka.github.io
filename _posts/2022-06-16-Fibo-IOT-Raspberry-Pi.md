---
layout: post
title:  "Fibo IOT Raspberry Pi"
date:   2022-06-16 00:12:24 +0100s
---

## List of contents:
1. Docker
2. Body

### Docker
#### Docker Desktop
Docker Desktop is an easy-to-install application for your Mac or Windows environment that enables you to build and share containerized applications and microservices. Docker Desktop includes Docker Engine, Docker CLI client, Docker Compose, Docker Content Trust, Kubernetes, and Credential Helper.

#### Docker Engine
Docker Engine is an open source containerization technology for building and containerizing your applications.

#### Docker Compose
Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration. To learn more about all the features of Compose, see the list of features.

Define your app’s environment with a Dockerfile so it can be reproduced anywhere.

Define the services that make up your app in docker-compose.yml so they can be run together in an isolated environment.

Run docker compose up and the Docker compose command starts and runs your entire app. You can alternatively run docker-compose up using the docker-compose binary.

#### Example

{% highlight ruby %}
version: "3.9"  # optional since v1.27.0
services:
  web:
    build: .
    ports:
      - "8000:5000"
    volumes:
      - .:/code
      - logvolume01:/var/log
    links:
      - redis
  redis:
    image: redis
volumes:
  logvolume01: {}
{% endhighlight %}

#### Compose vs Kubernetes
Kubernetes and Docker Compose are both container orchestration frameworks. Kubernetes runs containers over a number of computers, virtual or real. Docker Compose runs containers on a single host machine. 

#### Useful commands
{% highlight ruby %}

docker build .
docker images
√ generator % docker images --all                   
REPOSITORY   TAG              IMAGE ID       CREATED         SIZE
<none>       <none>           7033357bd792   9 minutes ago   173MB
rabbitmq     3.9-management   ff687db22397   9 days ago      221MB

docker run -d rabbitmq:3.9-management 
Unable to find image 'rabbitmq:3.9-management' locally
3.9-management: Pulling from library/rabbitmq

docker ps
{% endhighlight %}

#### Docker kill
The docker kill subcommand kills one or more containers. The main process inside the container is sent SIGKILL signal (default), or the signal that is specified with the --signal option. You can reference a container by its ID, ID-prefix, or name.

#### Docker stop [from my team]
The main process inside the container will receive SIGTERM, and after a grace period, SIGKILL. The first signal can be changed with the STOPSIGNAL instruction in the container’s Dockerfile, or the --stop-signal option to docker run

### UART
UART (Universal Asynchronous Transmitter Receiver), this is the most common protocol used for full-duplex serial communication. Designed to perform asynchronous communication

It requires a single wire for transmitting the data and another wire for receiving.

### Yocto
The Yocto Project is an open source collaboration project that helps developers create custom Linux-based systems for embedded products, regardless of the hardware architecture.

The project provides a flexible set of tools and a space where embedded developers worldwide can share technologies, software stacks, configurations and best practices which can be used to create tailored Linux images for embedded devices.

https://www.yoctoproject.org/software-overview/

Recipe: A recipe describes where you get source code and which patches to apply. They are stored in layers.

Layer: A collection of related recipes.

Build System - "Bitbake": a scheduler and execution engine which parses instructions (recipes) and configuration data.

Image

### NFC

{% highlight ruby %}
Waiting for a Tag/Device...

        NFC Tag Found

        Type :         'Type A - Mifare Ul'
        NFCID1 :        '04 62 6A D2 9C 39 81 '
                Record Found :
                                NDEF Content Max size :     '868 bytes'
                                NDEF Actual Content size :     '13 bytes'
                                ReadOnly :                      'FALSE'
                                Type :                 'Text'
                                Lang :                 'en'
                                Text :                 'jebaka'


                13 bytes of NDEF data received :
                D1 01 09 54 02 65 6E 6A 65 62 61 6B 61 

        NFC Tag Lost

Waiting for a Tag/Device...
{% endhighlight %}

{% highlight ruby %}
    g_TagCB.onTagArrival = onTagArrival;
    g_TagCB.onTagDeparture = onTagDeparture;
    nfcManager_doInitialize();
    nfcManager_registerTagCallback(&g_TagCB);
    nfcManager_enableDiscovery(DEFAULT_NFA_TECH_MASK, 0x01, 0, 0);
{% endhighlight %}



{% highlight ruby %}
{% endhighlight %}

## Mutex
The mutex class is a synchronization primitive that can be used to protect shared data from being simultaneously accessed by multiple threads.

## Semaphore
Semaphores are very useful in process synchronization and multithreading

### RabbitMQ
Is a message broker: it accepts and forwards messages. You can think about it as a post office: when you put the mail that you want posting in a post box

## RPC
In distributed computing, a remote procedure call (RPC) is when a computer program causes a procedure (subroutine) to execute in a different address space (commonly on another computer on a shared network), which is coded as if it were a normal (local) procedure call, without the programmer explicitly coding the details for the remote interaction.

## AMQP
The Advanced Message Queuing Protocol (AMQP) is an open standard application layer protocol for message-oriented middleware. The defining features of AMQP are message orientation, queuing, routing (including point-to-point and publish-and-subscribe), reliability and security.[1]