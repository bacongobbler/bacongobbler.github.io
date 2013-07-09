---
layout: post
title:  "My blog deployment workflow!"
date:   2013-07-09
tags: jekyll blog deployment
comments: true
---

My website lives within a git repository on [my private git site](http://git.bacongobbler.com/), and it was
made using [jekyll][jekyll]. This shows how I set up a remote repository on my deployment server for making
blog deployments as easy as <code>git push deploy master</code>.

# My Local Repository

Assuming that you're creating a new jekyll site from scratch and saving it to [GitHub][github], here would be my
typical workflow environment:

    $ git clone git@github.com:user/blog.git
    $ jekyll new blog
    $ git commit -am "first commit"
    $ git push master

At this point, we have a working Jekyll skeleton, where we can build the site. Technically, we could ssh into our hosting server, install git, ruby and jekyll, clone the repository, build the site, and place the folder into your web directory. That would take a lot of development time away if we needed to do that every time we wanted to test our site when we add new features, however. Why not automate this process with git post-receive hooks?

# Setting up post-receive hooks

I assume that the website will live on a server to which you have ssh access, and that things are set up so that you can ssh to it without having to type a password (i.e., that your public key is in ~/.ssh/authorized_keys on your remote server).

On the server, create a bare repository on your remote server to help deploy your local one on that site:

    $ ssh user@server.com
    $ mkdir blog.git && cd blog.git
    $ git init --bare

Create the public HTML folder to store your website and give access rights to the user that will be running the hook:

    $ sudo mkdir /var/www/www.server.com
    $ sudo chown user:user $_

'$_' grabs the last argument from the last command, which would be '/var/www/www.server.com'.
Now, start defining your post-receive hook to deploy your app:

```
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
```

Aside: The post-receive hook can receive multiple branches at once (for example if someone does a git push --all), so we also need to wrap the read in a while loop.

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

With this setup, if I try to push the 'staging' branch to deploy using <code>git push deploy staging</code>, it will checkout the staging branch, and deploy to my staging server, which is an exact duplicate of my production server, only that it may have edits that I have not added so that I may edit it freely without messing with my production server. When the changes are completed and ready, I'll merge the staging branch into the master branch, and run <code>git push deploy master</code> to deploy to my production server. Cool, huh?

[jekyll-gh]: https://github.com/mojombo/jekyll
[jekyll]:    http://jekyllrb.com
[github]:    http://github.com
