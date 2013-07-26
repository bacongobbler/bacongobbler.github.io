---
layout: post
title:  Working with Multiple Dockerfiles
date:   2013-07-26
tags: docker linux cloud
comments: true
---

[Dockerfiles][dockerfile] are a simplistic way to create a repeatable workflow for creating [Docker][docker] images.
To build an singular image, creating a description file called 'Dockerfile' will enable you to build these images.
When you've created the Dockerfile that you want to save, you can do so by running 'docker build .'. But what
if you want to have multiple Dockerfiles in one folder (eg. so you can deploy multiple docker images)?

Currently, 'docker build' cannot read from a file. However, it can read from STDIN. We can use this to our
advantage by running:

    docker build - < filename

This allows docker to read from 'filename'. For example, suppose that I want to hold dockerfiles for all
available services that my application may need. In this example, let's assume that I need MongoDB as
my NoSQL database, memcached for caching data for my app, and elasticsearch to quickly index and search
through a filter of several documents on MongoDB. For this, I would need three dockerfiles, right? So,
let's see this in action:

    bacongobbler@workbox:~$ mkdir -p project/dockerfiles
    bacongobbler@workbox:~$ cd $_
    bacongobbler@workbox:~/project/dockerfiles$ touch elasticsearch.dock mongodb.dock memcached.dock
    bacongobbler@workbox:~/project/dockerfiles$ # edit files to make legit dockerfiles
    bacongobbler@workbox:~/project/dockerfiles$ cat memcached.dock 
    from        ubuntu
    maintainer  Matthew Fisher <me@bacongobbler.com>

    run         apt-get update
    run         apt-get install -y memcached

    expose      11211

    cmd         ["memcached", "-u", "daemon"]
    bacongobbler@workbox:~/project/dockerfiles$ docker build -t bacongobbler/memcached - < memcached.dock
    bacongobbler@workbox:~/project/dockerfiles$ # docker build output. Afterwards, let's save it on the Docker Index
    bacongobbler@workbox:~/project/dockerfiles$ docker push bacongobbler/memcached
    bacongobbler@workbox:~/project/dockerfiles$ # rinse and repeat

If you're more interested in the docker project, go check out [http://docker.io/][docker] for more information.

Hope this helps someone out there!

[dockerfile]:   http://docs.docker.io/en/latest/use/builder/
[docker]:       http://docker.io/
