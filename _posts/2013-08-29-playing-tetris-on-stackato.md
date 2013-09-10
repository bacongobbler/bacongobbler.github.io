---
layout: post
title:  I heard you like Tetris...
date:   2013-08-29
tags: games nodejs stackato devops
comments: true
---

So why not play tetris inside Stackato, or any other server that allows you to
tunnel via SSH? Now you can, thanks to the help of 
[Mathias Buus](https://github.com/mafintosh) 
who created a program that will let you play tetris in full colour in your
terminal window.

Most of my time at ActiveState HQ is spent on getting open source
applications like [tetris](https://github.com/Stackato-Apps/tetris) or 
[docker's internal registry](https://github.com/Stackato-Apps/docker-registry)
'stackatofied'. We choose applications that are new and exciting or that
demonstrate deploying an application in some way. Deploying a Node.js
application is a whole lot different than deploying a Grails or a Python
app, and we try to show developers how to approach each deployment process
so that they can end up with a few more hairs at the end of the day. Even a
Rails app's deployment process could be completely different from another's
simply because it used a different gem or if it uses custom rake tasks to help
the deployment process!

With the Tetris application, we wanted to show how someone can deploy an
application to Stackato that does not have a web interface, or a display at
all! In fact, this application is just a module I found on npm. What I thought
was completely cool was how this application could also work through an SSH
tunnel... Which got me thinking (duck for cover, Matt's thinking again!).

Stackato has a feature where you can SSH directly into a running application.
These applications are running inside Linux Containers, or LXCs for short. In
our Stackato client, you can SSH directly into the container, allowing you to
directly run applications in the container, edit files, and potentially patch
your application on the fly (not that I'm recommending that, but it's an
option). You can do this by running:

    $ stackato ssh <appname>

ssh allows you to also run commands over that ssh tunnel, like...

    $ stackato ssh "uptime"

This command lets you see how long that container has been running for.

Armed with this knowledge, let's play Tetris over SSH!!

    $ git clone https://github.com/Stackato-Apps/tetris
    $ cd tetris
    $ stackato push -n tetris
    # go get a beer! Tetris is much better with beer.
    $ stackato ssh "node tetris"

Standard keyboard commands apply. "up" to rotate the block, "left" and "right"
to move the current block, "down" to make the block go down faster.

Have fun playing Tetris on Stackato!
