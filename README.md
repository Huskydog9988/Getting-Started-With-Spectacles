<a href="https://spec.pleb.xyz/"><img src="https://raw.githubusercontent.com/Huskydog9988/Getting-Started-With-Spectacles/main/spectacles.svg" alt="Spectacles"></a>

# Getting Started With Spectacles

A more comprehensive guide to the Spectacles ecosystem.

[Offical site](https://spec.pleb.xyz/)<br>
[Offical Github](https://github.com/spec-tacles/)

## Table of Contents

- [What is Spectacles?](#What-is-Spectacles?)
- [Why Should You Care?](#Why-Should-You-Care?)
- [When Shouldn't You Use Spectacles?](#When-Shouldn't-You-Use-Spectacles?)
- [What To Expect](#What-to-Expect)
- [Prerequisite Software](#Prerequisite-Software)
- [1. Getting Started](#1.-Getting-Started)
- [2. Understanding The Different Services](#2.-Understanding-The-Different-Services)
- [JS Library Quirks](#JS-Library-Quirks)

## Chapters

### What is Spectacles?

Spectacles is a distributed Discord API wrapper through the use of different applications and libraries aims to make stable, microservice-oriented Discord bots.

### Why Should You Care?

You should care because Spectacles makes horizontally scalable discord bots easier to create, deploy, and manage.

### When Shouldn't You Use Spectacles?

The only reasons you shouldn't use this libray is if every single piece of this guide blows over your head, or if your just creating a super basic bot for you and your friends.

### What to Expect

You should expect very little code from this guide. The little code given will be in JS. If you don't know JS, this wrapper has libraries in many differant langs so don't worry about not knowing it.
You should also expect very little explination as to what every system outside of Spectacles is doing.
For example, you will be using Docker and Docker Compose. You aren't expected to know every little tibit, but it is expect that you understand in general how they work.
Another example is RabbitMQ, since it isn't a service you will be directly interacting with, it isn't expected that you know anything, but you should know the basics.
The purpose of this guide is to help you understand how Spectacles as a whole works, not to teach you every system involved with it.
If you wish to learn more about the these other systems, there are far better guides out there than what I could ever write.

### Prerequisite Software

To make this entire process easier, and more managable, [Docker](https://docker.com/) is going to be used to containerize all of the software used. The following general steps will install all of the software you are going to need to get started.

1. Visit [Docker's get started](https://www.docker.com/get-started) page and install the Docker Desktop app.
2. [Install Docker Compose](https://docs.docker.com/compose/install/) for your system.
3. If you wish to follow along with the minimal code previded, you should have [Node.js installed](https://nodejs.org/en/download/).

### 1. Getting Started

To make things simple, a docker compose file (`docker-compose.yml`) is recommended to help manage all of the docker contianers.
The following compose file is an example of a what such a file in this ecosystem could look like.

```yaml
version: "3.7"

services:
  rabbitmq:
    image: rabbitmq:3-management-alpine
    hostname: "rabbitmq" # Leave as is
    labels:
      com.naval-base.description: "RabbitMQ Broker" # Leave as is
    expose:
      # Leave as is
      - "4369"
      - "5671"
      - "5672"
      - "25672"
      - "15671"
      - "15672"
    # restart: unless-stopped
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 60s
      timeout: 5s

  redis:
    image: redis:5-alpine
    labels:
      com.naval-base.description: "Redis" # Leave as is
    expose:
      # Leave as is
      - "6379"
    # restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping"]
      interval: 10s
      timeout: 5s

  gateway:
    image: spectacles/gateway:latest
    labels:
      com.naval-base.description: "Gateway Ingress" # Leave as is
    environment:
      # More config can be found here: https://github.com/spec-tacles/gateway
      DISCORD_TOKEN: "" # Discord bot token
      DISCORD_EVENTS: "READY,MESSAGE_CREATE,GUILD_CREATE" # A comma seperated array of events
      BROKER_TYPE: "amqp" # only supported type; any other value sends/receives from STDIN/STDOUT
      BROKER_GROUP: "gateway" # leave as is
      BROKER_MESSAGE_TIMEOUT: "10m"
      # PROMETHEUS_ADDRESS: ":8080" # Address to prometheus if you are using it
      # PROMETHEUS_ENDPOINT: "/metrics" # prometheus endpoint
      AMQP_URL: "amqp://rabbitmq" # URL to AMQP, don't change unless needed
      SHARD_STORE_TYPE: "redis" # Only supported type
      SHARD_STORE_PREFIX: "gateway" # String to prefix shard-store keys
      REDIS_URL: "redis://redis:6379"
      REDIS_POOL_SIZE: 5 # size of Redis connection pool
    expose:
      - "8080"
    # restart: unless-stopped

  proxy:
    image: spectacles/proxy:latest
    labels:
      com.naval-base.description: "Discord proxy" # Leave as is
    environment:
      RUST_LOG: info # leave as is
      TIMEOUT: ""
      REDIS_URL: "redis://redis:6379"
      AMQP_URL: "amqp://rabbitmq" # URL to AMQP, don't change unless needed
      AMQP_GROUP: "rest" # leave as is
      AMQP_EVENT: "REQUEST"
    # restart: unless-stopped
```

(Credit to Navel Base for making this compose file [here](https://github.com/Naval-Base/yuudachi/))

### 2. Understanding The Different Services

Time to explain what all those services above in the compose file are doing.
RabbitMQ is the message broker all the other major parts of the system are going to be communicating with. If that description is confusing, think of RabbitMQ as a conductor, and all the the other servies as instruments in a orchestra.

Redis, if you somehow don't know what it is, is an "in-memory data structure store, used as a database, cache and message broker." Basicly, Spectacles thus far is using Redis as a place to store different information about the different shards of your bot.

The Gateway service is, as the name implies, the gateway to discord. To quote the Gateway repo "The Spectacles gateway acts as a standalone process between your Discord bot application and the Discord gateway, allowing your bot to focus entirely on application logic." So in essence the code you end up writing is like that of most libraries. The main differance being that instead of it all happeing in the same process, its happening in different process.

If you are wondering what other events you can subscribe to besides the ones listed in the gateway enviroment look at the Discord dev docs.
