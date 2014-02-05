---
layout: post
title:  Blog Deployment Workfow using Nginx, Jekyll, and git post-receive Hooks
date:   2013-07-09
author: Matthew Fisher
tags: jekyll blog deployment
comments: true
---

This blog post shows how I set up a remote repository on my deployment server for making
blog deployments as easy as <code>git push deploy master</code>. This website lives within a git repository on my private git site, and it was
made using [jekyll][jekyll]. 

# Prerequisites

There are a couple things that are required before you start this tutorial. You must have:

* access to a private server for hosting your site (if it's just for testing purposes, a laptop or computer will do)
* basic knowledge of git. There is a good 15-minute tutorial on git [here][git-tut]
* basic knowledge of web servers and how they work (not required)
* shell scripting knowledge in BASH (sorry Windows users, no tutorial for you!)
* knowledge of the Markdown language and [jekyll][jekyll] (if you're familiar with READMEs on Github or you comment on Reddit, then you already know Markdown).

# Starting from scratch

Assuming that you're creating a new jekyll site from scratch and saving it to [GitHub][github], here would be my
typical workflow environment:

    $ git clone git@github.com:user/blog.git
    $ jekyll new blog
    $ git commit -am "first commit"
    $ git push master

At this point, we have a working Jekyll skeleton, where we can build the site. Technically, we could ssh into our hosting server, clone the repository, build the site, and place the folder into your web directory. That would take a lot of development time away if we needed to do that every time we wanted to test our site when we add new features, however. Why not automate this process with git post-receive hooks?

# Setting up post-receive hooks

I assume that the website will live on a server to which you have ssh access, and that things are set up so that you can ssh to it without having to type a password (i.e., that your public key is in ~/.ssh/authorized_keys on your remote server).

On the server, create a bare repository to help deploy your blog on that site:

    $ ssh user@server.com
    $ mkdir blog.git && cd blog.git
    $ git init --bare

Create the public HTML folder to store your website and give access rights to the user that will be running the hook:

    $ sudo mkdir /var/www/blog.server.com
    $ sudo chown user:user $_

'$_' grabs the last argument from the last command, which would be '/var/www/www.server.com'.
Now, start defining your post-receive hook to deploy your app:

    bacongobbler@bacongobbler:~/blog.git/hooks$ cat post-receive 
    #!/bin/sh
    GIT_REPO=git@git.bacongobbler.com:bacongobbler/blog.git
    TMP_GIT_CLONE=/tmp/blog
    PUBLIC_WWW=/var/www/blog.bacongobbler.com
    STAGING_WWW=/var/www/staging.bacongobbler.com

    while read oldrev newrev refname
    do
        branch=$(git rev-parse --symbolic --abbrev-ref $refname)

        # We only want to deal with the master and staging branches
        if [ "$branch" = "master" ]; then
            git clone -b master $GIT_REPO $TMP_GIT_CLONE
            jekyll build -s $TMP_GIT_CLONE -d $PUBLIC_WWW
        elif [ "$branch" = "staging" ]; then
            git clone -b staging $GIT_REPO $TMP_GIT_CLONE
            jekyll build -s $TMP_GIT_CLONE -d $STAGING_WWW
        fi

        rm -rf $TMP_GIT_CLONE
    done

Since a post-receive hook can receive multiple branches at once (for example if someone does a git push --all), we need to wrap the read in a while loop. You'll also need to install git, Ruby, and Jekyll on your server in order for this script to run.

Make sure the post-receive hook is executable:

    $ chmod +x hooks/post-receive

Then, back on your local workstation, add the new remote mirror:

    $ git remote add deploy user@server.com:~/blog.git
    $ git push deploy master

And when you want to update your website again, run:

    $ git push deploy master

# What about pushing to a Staging server?

You may have noticed these few lines that I added to my post-receive hook:

        elif [ "$branch" = "staging" ]; then
            cd $TMP_GIT_CLONE && git checkout staging
            jekyll build -s $TMP_GIT_CLONE -d $STAGING_WWW
        fi

With this setup, if I try to push the 'staging' branch to deploy using <code>git push deploy staging</code>, it will checkout the staging branch and deploy to my staging server, which is an exact duplicate of my production server. The only difference between the two is that it may have commits that I have not yet added to production so that I may edit it freely without messing with my production server. When the changes are completed and ready, I'll merge the staging branch into the master branch, and run <code>git push deploy master</code> to deploy to my production server. Cool, huh?

If you want to use this workflow as well, you'll need a folder to store your staging site:

    $ sudo mkdir /var/www/staging.server.com
    $ sudo chown user:user $_

# Nginx configuration

Now that we have our blog uploaded to our deployment server, we need a web server to serve those files over the web! To do this on Ubuntu/Mint/Debian, run:

    $ sudo apt-get update

    # optional, if your server needs upgrading
    $ sudo apt-get upgrade

    $ sudo apt-get install nginx

Then, create the files for each site; one for staging, and one for deployment. Just remove staging if you just want a deployment server, and don't forget to change my name with your server name:

    $ cd /etc/nginx/sites-available/
    $ cat blog.bacongobbler.com
    server {
        root /var/www/blog.bacongobbler.com;
        index index.html;
        server_name blog.bacongobbler.com;

        # go ahead, try to 404 my site. I dare you.
        error_page 404 /404.html;
    }

    $ # these sites are exact duplicates, remember? Save time and just copy it!
    $ cp blog.bacongobbler.com staging.bacongobbler.com

    $ # enable them with nginx
    $ cd ../sites-enabled/
    $ ln -s /etc/nginx/sites-available/blog.bacongobbler.com .
    $ ln -s /etc/nginx/sites-available/staging.bacongobbler.com .

    $ # restart nginx
    $ sudo service nginx restart

And now your site sould now be up and running! Enjoy your automated deployment workflow!

Please feel free to share your experiences or troubles with this tutorial below, and I'll be happy to assist you. If I have missed anything, please send me an email. :)

[jekyll-gh]:    https://github.com/mojombo/jekyll
[jekyll]:       http://jekyllrb.com
[github]:       http://github.com
[git-tut]:      http://try.github.io/
