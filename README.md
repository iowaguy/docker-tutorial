# Docker Tutorial

This tutorial will walk you through using Docker for assigned projects in CS 4730: Distributed Systems. Credit for most of this write-up goes to [Asad Salman](https://github.com/asadsalman/docker-tutorial).

## What is Docker?

Docker is a system for building lightweight containers. A container is an isolated environment that runs inside a host machine. It is conceptually similar to a virtual machine (VM), but its internals are quite different. A docker *container* is built from a docker *image*—an image is sort of a blueprint for a container similar to how a class is a blueprint for an object in object-oriented languages.

A Docker image is built in layers with each layer either placing files into the image filesystem, executing commands, or defining attributes for the eventual containers that will be built. You can spin up any number of containers with the same image (though these each use system resources, so if you spin up enough, eventually your computer will complain).

## Why use Docker?

Docker can be a useful tool for a number of reasons. Some reasons of interest to us (in this class) are:

1. Project submissions become highly reproducible, thus less likely to be graded incorrectly
2. It is much easier to write and test out distributed applications for the course, which will likely have multiple instances running, once you get past the initial learning curve of Docker

## Installing Docker

Install the latest stable release of Docker Community Edition on your computer. You will need superuser/adminstrator access for this so it won’t work on Northeastern Machines. You can get further instructions on [Docker’s website](https://docs.docker.com/install/) for your specific operating system.

## Simple application

I’ll walk you through writing a simple C “hello world” application in Docker. Later on in the tutorial, we’ll get into networking in Docker.

In your local workspace, create a directory `docker-hello` by running `mkdir docker-hello` and create the following two files **in that directory** using the `touch` command: `hello.c` and `Dockerfile`.

Open the `hello.c` file in your text editor, and copy the following simple “hello world” application (written in C) into that file:

```c
#include <stdio.h>

int main(){
    printf("hello world");
}
```

Save and close `hello.c`.

Next, open `Dockerfile` in your editor and copy in the following:

```docker
FROM ubuntu:22.04

RUN apt-get update
RUN apt-get install -y gcc

ADD hello.c /app/
WORKDIR /app/
RUN gcc hello.c -o hello

ENTRYPOINT /app/hello
```

While there are many, many variations of how this Dockerfile could look like and still perform the same task, this is a good, simple one for our purposes. I’ll walk through what each line means.

The first line, `FROM ubuntu:latest`, specifies what base image we’re using for our container. 

There are many options here (to see some others, checkout [DockerHub](https://hub.docker.com/)), I usually go with `ubuntu:latest` and build on top of that. I suggest using an OS you are already familiar with.

On the next line, we update the Ubuntu package repository and then install `gcc`, the C compiler. We’ll use `gcc` to compile `hello.c` (alternatively you could have used a base image that already includes gcc). 

In the fourth code line, we copy over the `hello.c` file from the local filesystem into the image filesystem using the `ADD` directive.

Next, using the `WORKDIR` directive, we move to the `/app/` directory in the image filesystem (similar to using the `cd` command). We then compile `hello.c` using `gcc`.

Finally, I set the entrypoint of our container to be the newly built `hello` binary. What this means is, as soon as the container runs, `/app/hello` will be executed.

Let’s now build the image and run the container. While in the `docker-hello` directory, run the following command in the shell:

```bash
docker build . -t "dockerhello"
```

This builds the image using the Dockerfile (defined above) in the current directory, and gives it the tag “dockerhello”. This may take a few minutes if you are downloading the base image for the first time. Now we can instantiate an instance of the `dockerhello` container with:

```bash
docker run dockerhello
```

You should see “hello world” printed on the screen.

## Networking demo in Docker

In order to have multiple Docker containers talking to each other, we are going to create a user-defined bridge network in Docker. While containers can communicate over the default bridge network in Docker, the bridge does not provide container name to IP address resolution. This name resolution is useful because, while writing your distributed application, you can make a peer list of hostnames once and use it between different runs of your system without having to update it, as long as you’re using the same container names between runs.

Exit the `docker-hello` directory (`cd ..`), and create a new directory called `docker-network` and `cd` into it.

To start off, we list the existing Docker networks using:

```bash
docker network list
```

Next we’re going to add a Docker network:

```bash
docker network create --driver bridge mynetwork
```

Now to create a new Dockerfile that will let us play around with networking between containers:

```docker
FROM ubuntu:22.04

RUN apt-get update
RUN apt-get install -y netcat iputils-ping
```

Put the above Dockerfile in a new directory called `docker-network`. We will be using `netcat`, the TCP/IP Swiss Army Knife, to test out networking between Docker containers. `netcat` comes in very handy when testing out networking applications and I’d strongly recommend getting comfortable with it for this course.

Next, navigate to `docker-network` and build the Docker image:

```bash
docker build . -t dockernetwork
```

Now we’re going to run two container instances of the image `dockernetwork` and have them talk to each other. These containers will be part of the `mynetwork` we previously created.

In a terminal, run

```bash
docker run -it --name first --network mynetwork dockernetwork
```

The `-it` flags tell docker that the container should be interactive and accept `tty` input, which just means that you can interact with it from the terminal. You’ll notice that we did not specify an `ENTRYPOINT` in this container. The default entrypoint for the Ubuntu image is `/bin/bash`, which means that it will run a bash shell upon startup. So when you start an interactive container, it will drop you into a the bash shell. Additionally, you can provide arguments after the image name which will be passed to the entrypoint.

In another terminal (on your host machine), run:

```bash
docker run -it --name second --network mynetwork dockernetwork
```

You will now have two running instances of Docker containers based on the `dockernetwork` image, both part of `mynetwork` network.

We will now start a TCP server that accepts connections on port 3000 in the `first` container. In the terminal where you started the `first` container, run:

```bash
netcat -nvlp 3000
```

From the terminal where you started the `second` container, connect to the `first` container by running:

```bash
netcat -v first 3000
```

At this point, both containers should show that you have successfully established a TCP connection. Anything you type now in the `second` container’s shell, followed by the enter/return key, will be echoed on the other side, indicating a successful TCP connection.

# Docker Compose
[Docker Compose](https://docs.docker.com/compose/) is a container orchestrator
for Docker. Installation instructions can be found
[here](https://docs.docker.com/compose/install/). Docker Compose lets you
statically define attributes and arguments for your containers in a YAML
configuration file. For example, the following connects two containers over a
bridge network, and then has each of them print a short message before exiting.

``` yaml
services:
  one:
    image: dockernetwork
    networks:
      - mynetwork
    hostname: "host1"
    command: echo "hello, host2"

  two:
    image: dockernetwork
    networks:
      - mynetwork
    hostname: "host2"
    command: echo "hello, host1"

networks:
  # The presence of these objects is sufficient to define them
  mynetwork: {}
```

To run Docker Compose, you can use the following

``` bash
docker compose -f <COMPOSE_FILENAME.yml> up
```

Which, for the above compose file would yield the following output

``` text
[+] Running 2/2
 ⠿ Container lab3-two-1  Recreated                                                                                                                                                                                                        8.7s
 ⠿ Container lab3-one-1  Recreated                                                                                                                                                                                                       19.0s
Attaching to lab3-one-1, lab3-two-1
lab3-one-1  | hello, host2
lab3-two-1  | hello, host1
lab3-one-1 exited with code 0
lab3-two-1 exited with code 0
```

# Troubleshooting
If you are seeing the error `No address associated with hostname`, it may be caused by a DNS issue. Try adding a well-known public resolver like Google's `8.8.8.8` or Cloudflare's `1.1.1.1` to `~/.docker/daemon.json` or `/etc/docker/dameon.json`. For example,

```json
{
  "dns": ["10.0.0.2", "1.1.1.1"]
}
```
Note that you may need to restart the Docker service if you are running one.

# Acknowledgements

*********This write-up was heavily adapted from [Asad Salman's](https://github.com/asadsalman/docker-tutorial).*
