---
layout: post
title:  Deis on DigitalOcean
date:   2013-09-24
tags: deis devops paas
comments: true
---

Deploying applications to web servers on your own infrastructure these days has never been easier. Gone are the days of having to manually install packages, run bootstrapping scripts, replicate production environments, and document the entire process so that you can successfully run the same job at 3AM because your application went down. Thanks to new emerging technologies like Configuration Management tools, Platform as a Service layers, and the push for companies to adopt a DevOps approach to their solutions, IT orchestration and application management got a hell of a lot easier. One of these new tools that leverage both containerization and configuration management in their workflow is [Deis][deis]. As of [commit 1ca8d840f5](https://github.com/opdemand/deis/commit/1ca8d840f5b655a269370aa9d7a8fe8e7c8da577) at https://github.com/opdemand/deis, DigitalOcean clustering support has been added, so you can have your own mini-Heroku on DigitalOcean! The documentation for getting your own cluster set up on DigitalOcean can be found [here](http://docs.deis.io/en/latest/installation/digitalocean/). This post will (hopefully!) help give you a basic overview on how easy it is to deploy a cluster on DigitalOcean.

[Deis][deis] (pronounced DAY-iss) is an open source Application Platform that is designed to work with both public and private clouds. You get all the benefits of a Heroku-styled workflow for your users, all while gaining the flexibility of an IT automation tool. It is an open source PaaS that makes it easy to deploy and scale LXC containers and Chef nodes used to host applications, databases, middleware and other services. Deis leverages [Chef][chef], [Docker][docker] and [Heroku Buildpacks](https://devcenter.heroku.com/articles/buildpacks) to provide a private PaaS that is lightweight and flexible to your needs.

# Setting up a Cluster on DigitalOcean

Starting from complete scratch, let's bring up a [controller][controller], and then a three-node [formation](http://docs.deis.io/en/latest/gettingstarted/terms/formation/) on DigitalOcean.

## Set up your Account

 First, [let's create an account at DigitalOcean](https://www.digitalocean.com/?refcode=d6bd7649dc9d). If you sign up, please use this referral code! I scratch your back, you scratch mine. ;)

### Create a Snapshot

Next, we are going to create a snapshot. This snapshot is just a base Ubuntu 12.04.3 LTS image with a couple of added dependencies for faster boot times. We need 12.04.3 for the linux 3.8 kernel, which is required to run Docker.

First, spawn up a new droplet with the following parameters:

    hostname: any (I typically use "deis-snapshot", but the name is arbitrary)
    size: 512MB
    region: any
    image: Linux Distributions - Ubuntu - Ubuntu 12.04.3 x64
    *optional* sshkey: an SSH key you want to use with this instance (will be removed later)
    virtio: enable (not important, but is a nice perk)

After the droplet has booted up and is available, start running the commands necessary to prepare the snapshot:

    $ ssh root@<ip-address>
    # apt-get install -qy git
    # git clone https://github.com/opdemand/deis.git
    # cd deis/contrib/digitalocean
    # bash prepare-digitalocean-snapshot.sh
    # cd $HOME && rm -rf deis
    # shutdown now

Then, from the console, shut down the droplet, and then take a snapshot of it, naming the snapshot "deis-base". Naming is important here, as the controller script and the DigitalOcean provider library looks for this specific snapshot name.

After you've created the snapshot, we will want to distribute that snapshot across all regions, so that we can create formations in any region. For that, go into the DigitalOcean control panel and go to the Images tab. Click on the globe icon for the deis-base snapshot to distribute it to all regions.

## Boot up a Controller Node

Now, let's bring up a Deis [controller][controller]. Documentation for this is also [available at docs.deis.io](http://docs.deis.io/en/latest/installation/digitalocean/).

### Prerequisites

To start, you'll need git, RubyGems, and an account on a Chef server accessible with the knife command. chef-solo deployments are not supported, but you can create a free account at https://manage.opscode.com, which will let you bring up to 5 nodes before you have to start paying. Either that, or you can host your own chef server. The choice is yours.

### Clone the Deis Repository

    git clone https://github.com/opdemand/deis.git 

### Install Dependencies

    cd deis && bundle install && pip install -r requirements.txt

### Configure knife

The knife-digital_ocean plugin needs some configuration before we get started. For example, here is my knife.rb:

    $ cat ~/.chef/knife.rb
    current_dir = File.dirname(__FILE__)
    log_level                :info
    log_location             STDOUT
    node_name                "bacongobbler"
    client_key               "#{current_dir}/bacongobbler.pem"
    validation_client_name   "bacongobbler-validator"
    validation_key           "#{current_dir}/bacongobbler-validator.pem"
    chef_server_url          "https://api.opscode.com/organizations/bacongobbler"
    cache_type               'BasicFile'
    cache_options( :path => "#{ENV['HOME']}/.chef/checksums" )
    cookbook_path            '~/.berkshelf/cookbooks'
    knife[:digital_ocean_client_id] = ENV['DIGITALOCEAN_CLIENT_ID']
    knife[:digital_ocean_api_key]   = ENV['DIGITALOCEAN_API_KEY']

You can grab the client ID and the API key from the DigitalOcean control panel. Set them up in your bashrc file like so:

    $ echo "export DIGITALOCEAN_CLIENT_ID=xxxxxxxxx" >> ~/.bashrc
    $ echo "export DIGITALOCEAN_API_KEY=xxxxxxxxx" >> ~/.bashrc
    $ source ~/.bashrc

After that, you should be able to use knife:

    $ knife digital_ocean region list
    ID  Name
    1   New York 1
    2   Amsterdam 1
    3   San Francisco 1
    4   New York 2

### Provisioning a Controller

The DigitalOcean provisioning script expects one argument: the ID of the cloud region in which to host the controller. Take note of the output of the command above, then run the DigitalOcean provisioning script, which takes several minutes to complete:

    $ # upload cookbooks
    $ berks install && berks upload
    $ # workaround for DigitalOcean SSH issues
    $ rm ~/.ssh/known_hosts
    $ ./contrib/digitalocean/provision-digitalocean-controller.sh 4
    Provisioning a deis controller on DigitalOcean!
    Creating new SSH key: deis-controller
    + ssh-keygen -f ~/.ssh/deis-controller -t rsa -N '' -C deis-controller
    ...
    Created data_bag[deis-apps]
    Provisioning deis-controller with knife digital_ocean...
    + knife digital_ocean droplet create --bootstrap-version 11.6.2 ...
    Droplet creation for deis-controller started. Droplet-ID is 123456
    Waiting for IPv4-Address.done
    IPv4 address is: 198.199.96.159
    ...
    198.199.96.159 Chef Client finished, 74 resources updated
    198.199.96.159
    + set +x
    Please ensure that "deis-controller" is added to the Chef "admins" group.

Once the provisioning script finishes, a deis-controller client object will have been created on the Chef server. You must log in to the Opscode web portal (or its equivalent if you’re using a local Chef server) and add deis-controller to the admins group. This is required so the controller can add and delete node and client records when scaling nodes.

This is also the point at which you should set up name resolution for the Deis controller, unless you prefer to access it by IP address. So from here, we will refer to 198.199.96.159 as “deis.bacongobbler.com.”

Congratulations! You’ve set up a Deis controller, the heart of your new private PaaS. Now let’s connect to the controller and set it to work.

### Install the Deis Client

Install the Deis client with pip:

    $ pip install deis
    Downloading/unpacking deis
      Downloading deis-0.1.1.tar.gz
      Running setup.py egg_info for package deis
    ...
    Successfully installed deis
    Cleaning up...

### Register with the Controller

Registration will discover SSH keys automatically and use the environment variables DIGITALOCEAN_CLIENT_ID and DIGITALOCEAN_API_KEY to configure the DigitalOcean provider with your credentials.

    $ deis register http://deis.bacongobbler.com
    username: bacongobbler
    password:
    password (confirm):
    email: me@bacongobbler.com
    Registered myuser
    Logged in as myuser

    Found the following SSH public keys:
    1) id_rsa.pub
    Which would you like to use with Deis? 1
    Uploading /Users/myuser/.ssh/id_rsa.pub to Deis... done

    Found Digitalocean credentials: hkrVAMXXXXXXXXXXXXXXXX
    Import these credentials? (y/n) : y
    Uploading Digitalocean credentials... done

### Deploy a 3-node Cluster in San Francisco

Now that we have prepared our controller, let's deploy a multi-node cluster over in San Francisco.

    $ deis formations:create dev --domain=bacongobbler.com

The --domain command here is VERY IMPORTANT. This makes Deis resolve the formation's domain name to bacongobbler.com. Without this, we cannot host multiple applications on the same cluster.

Let's create two layers. One will be a proxy layer, and the other will host the applications: 

    $ deis layers:create dev proxy digitalocean-san-francisco-1 --proxy=y --runtime=n
    $ deis layers:create dev runtime digitalocean-san-francisco-1 --proxy=n --runtime=y

And now, let's provision some nodes on the layers:

    $ deis nodes:scale dev proxy=1 runtime=2
    Scaling nodes... but first, coffee!
    done in 402s

This will scale up and provision all three nodes in parallel, which is a big bonus for us.

### Set up Wildcard DNS

We will want all requests to go through the proxy layer, so we'll set up our wildcard subdomain over to the proxy.

### Deploy an application to the Cluster

So now that we have our cluster set up, let's spawn an application up on the server. I'll be pushing a Flask hello world application. To start, let's change directories to one of the apps and push it to the cluster:

    $ ls
    deis  example-python-flask
    (venv)bacongobbler@ziggs opdemand$ cd example-python-flask/
    (venv)bacongobbler@ziggs example-python-flask$ deis apps:create --formation=dev
    Creating application... done, created cubist-farmland
    Git remote deis added
    (venv)bacongobbler@ziggs example-python-flask$ git push deis master
    Delta compression using up to 2 threads.
    Compressing objects: 100% (42/42), done.
    Writing objects: 100% (74/74), 18.06 KiB | 0 bytes/s, done.
    Total 74 (delta 25), reused 74 (delta 25)
           Python app detected
    -----> No runtime.txt provided; assuming python-2.7.4.
    -----> Preparing Python runtime (python-2.7.4)
    -----> Installing Distribute (0.6.36)
    -----> Installing Pip (1.3.1)
    -----> Installing dependencies using Pip (1.3.1)
           [...]
           Cleaning up...
    -----> Discovering process types
           Procfile declares types -> web

    -----> Compiled slug size: 28.4 MB
           Launching... done, v2

    -----> cubist-farmland deployed to Deis
           http://cubist-farmland.bacongobbler.com

           To learn more, use `deis help` or visit http://deis.io

    To git@deis.bacongobbler.com:cubist-farmland.git
     * [new branch]      master -> master
    (venv)bacongobbler@ziggs example-python-flask$ curl http://cubist-farmland.bacongobbler.com
    powered by Deis!

In the quote "powered by Deis!", "Deis!" is pulled from a environment variable called POWERED_BY. We can change that on the fly.

    $ deis config:set POWERED_BY=bacongobbler --app=cubist-farmland
    === cubist-farmland
    POWERED_BY: bacongobbler
    (venv)bacongobbler@ziggs example-python-flask$ curl cubist-farmland.bacongobbler.com
    Powered by bacongobbler

### Scale an application

Having a running instance of an app is great, but what if we want to run multiple instances of the same app within that cluster?

    $ deis containers:scale web=3 --app=cubist-farmland
    Scaling containers... but first, coffee!
    done in 10s

    (venv)bacongobbler@ziggs example-ruby-sinatra$ deis containers:list --app=cubist-farmland
    === cubist-farmland Containers

    --- web: `gunicorn -b 0.0.0.0:$PORT app:app`
    web.1 up 2013-11-03T11:38:03.632Z (dev-runtime-1)
    web.2 up 2013-11-03T11:55:41.764Z (dev-runtime-2)
    web.3 up 2013-11-03T11:55:41.783Z (dev-runtime-2)

Awesome! now we have 3 instances of the 'web' entry in our Procfile.

Thank you so much for reading this howto! Please feel free to comment or ask any questions via IRC or email.

[controller]: http://docs.deis.io/en/latest/gettingstarted/terms/controller/
[chef]: http://www.opscode.com/chef/
[deis]: http://deis.io/
[docker]: http://docker.io/
