---
layout: post
title:  How to create your own Custom Service in Stackato
date:   2013-07-15
tags: stackato service
comments: true
---

Services are essential for cloud-enabled applications and therefore core to Stackato. Whether it's relational database services like [PostgreSQL][postgresql], message queueing services like [RabbitMQ][rabbitmq], NoSQL data stores like [MongoDB][mongodb], or caching with [memcached][memcached], our devs have worked out how to get popular services integrated with [Stackato][stackato], and the default services have been great for almost every application.

But what if you wanted to rely on your own custom service? Hopefully, by the end of this post, you'll understand how services work in Stackato, how they’re provisioned, and how to make your own custom service plugin for Stackato!

For the past few weeks on IRC, a number of customers have been asking us how to implement their own services on Stackato. Many of the ones I talked to were interested in [Elasticsearch][elasticsearch], a full-text indexing service for real-time search and analytics in the cloud. I took this on as a research project for myself, but also for customers who would look at this and say, "So this is how you implemented xyz!".

Stackato services are implemented through Ruby applications that act as adapters between services (which can be any executable that can run in the background) and Stackato's many interfaces. The relevant interfaces to Stackato are:

 * kato
 * doozer
 * supervisord
 * the cloud controller

The Ruby application for each service acts as a gateway between the service and the interfaces, as well as helping provision the services. Provisioning is a particularly broad term here in the services department. Let's break down some examples of provisioning:

 * creating a new database, along with a user and password to access that database 
 * spawning a new process, allocating a new port number
 * creating a new filesystem directory and mounting it in a container

The first model is most common with database services like PostrgeSQL and MySQL. The second, process-based provisioning, is done for services that run individual background daemons for each service instance (e.g. Memcached and MongoDB). Basically, each service maps to a separate process or database depending on the context, and exposes to the user/application the details and credentials for accessing the service instance.

# Let's get Crackin'!

Now that we have a basic understanding of services, let's create one!

Before moving on to the slightly more complex [Elasticsearch](https://github.com/ActiveState/stackato-elasticsearch-service) example, we'll start with the simpler [stackato-echoservice](https://github.com/ActiveState/stackato-echoservice) sample:

    git clone https://github.com/ActiveState/stackato-echoservice.git

Let's look at a few key areas within the Ruby project:

## Configuration

The config files for each service can be found in the aptly-named config folder:

    $ ls config/
    echo_gateway.yml  echo_node.yml

We notice that we have two configuration files: one for the gateway, and one for the nodes. A node is the provisioned service when you run <code>stackato add-service mongodb</code> or for whichever service you prefer. The gateway is the binary that kato and the other interfaces talk to when accessing data about the service and its nodes. Some points to note in each config are:

 * base_dir: where the service will be installed inside Stackato (which is typically within <code>/var/vcap/services/</code>)
 * port: the first port number that is provisioned to the first instance of this service. Increments by one for each subsequent service instance
 * cloud_controller_uri: the URI of the cloud controller or micro cloud
 * token: the token that is required to authenticate with kato.

Change these values as you see fit. Just make sure that you're not fighting for port numbers with another service!

## The Provisioner

The provisioner in most services are located at <code>lib/name_service/name_node.rb</code>. In most cases, it's there to run an executable, expose a port to that service, and save the credentials somewhere. For the echo service, it doesn't even run a separate executable. It just provisions a service, saves it to a local sqlite3 database (which is handled in <code>save_instance()</code>) and retrieves the credentials:

    def provision(plan, credential = nil, version=nil)
        instance = ProvisionedService.new
        if credential
            instance.name = credential["name"]
        else
            instance.name = UUIDTools::UUID.random_create.to_s
        end

        begin
            save_instance(instance)

        ... # exception handling
        end
        
        gen_credential(instance)
    end

For Elasticsearch, the provisioning code isn't that much different:

    def provision(plan, credentials = nil, db_file = nil)
        instance = ProvisionedService.new
        instance.plan = plan
        if credentials
            instance.name = credentials["name"]
            @free_ports_mutex.synchronize do
                if @free_ports.include?(credentials["port"])
                    @free_ports.delete(credentials["port"])
                    instance.port = credentials["port"]
                else
                    port = @free_ports.first
                    @free_ports.delete(port)
                    instance.port = port
                end
            end
        else
            @free_ports_mutex.synchronize do
                port = @free_ports.first
                @free_ports.delete(port)
                instance.port = port
            end
            instance.name = UUIDTools::UUID.random_create.to_s
        end

        begin
            instance.pid = start_instance(instance, db_file)
            save_instance(instance)
            @logger.debug("Started process #{instance.pid}")
        
        ... # cleanup code
        end

        gen_credentials(instance)
    end

The main difference between the two is that the Elasticsearch plugin uses a mutex when provisioning port numbers, and calls a new function called <code>start_instance()</code>, which runs <code>Process.spawn</code> on an executable on the server.

After tweaking it or after creating our own service, you can install the echo service by following the instructions in [the README.md][echo-readme]. However, if you're looking to install Elasticsearch, this has already been automated!

    $ git clone https://github.com/ActiveState/stackato-elasticsearch-service
    $ cd stackato-elasticsearch-service

Edit the 'cloud_controller_uri' to reflect - you guessed it - the Cloud Controller's URI.

    vim config/elasticsearch_gateway.yml

The following installs elasticsearch for the stackato user under */opt/elasticsearch* along with any dependencies elasticsearch relies on. If you want to install a newer/older version of Elasticsearch, change the VERSION variable to suit your needs.
    
    sudo ./scripts/install-elasticsearch.sh

To install the elasticsearch service to the current node:

    ./scripts/bootstrap.sh

Bootstrapping runs all of the commands shown in the echo service example, including installing the service gems, installing to supervisord and kato, loading into Doozer, adding the service AUTH token to the Cloud Controller, adding elasticsearch as a role, and finally restarting kato. After all this is done, you should be able to see "elasticsearch" as a useable service in the Stackato client.

    $ stackato services
    ============== System Services ==============
    
    +---------------+---------+------------------------------------------------+
    | Service       | Version | Description                                    |
    +---------------+---------+------------------------------------------------+
    | elasticsearch | 1.0     | Elasticsearch full-text searching and indexing |
    | filesystem    | 1.0     | Persistent filesystem service                  |
    | harbor        | 1.0     | External port mapping service                  |
    | memcached     | 1.4     | Memcached in-memory object cache service       |
    ...
    
    $ stackato create-service elasticsearch mysearch1
    Creating Service: OK
    $ stackato service mysearch1
    
    mysearch1
    +-------------+--------------------------------------+
    | What        | Value                                |
    +-------------+--------------------------------------+
    | credentials |                                      |
    | - host      | 192.168.69.32                        |
    | - hostname  | 192.168.69.32                        |
    | - name      | 6652315d-0f6b-48d6-8c88-5de7643a626c |
    | - node_id   | elasticsearch_node_1                 |
    | - port      | 9202                                 |
    |             |                                      |
    | email       | troyt@activestate.com                |
    | meta        |                                      |
    | - created   | Wed Jul 17 13:58:54 PDT 2013         |
    | - tags      | elasticsearch {bonsai cool}          |
    | - updated   | Wed Jul 17 13:58:54 PDT 2013         |
    | - version   | 1                                    |
    |             |                                      |
    | properties  |                                      |
    | tier        | free                                 |
    | type        | generic                              |
    | vendor      | elasticsearch                        |
    | version     | 1.0                                  |
    +-------------+--------------------------------------+

And that's how you add custom services to Stackato! If you have any questions about the whole process, please feel free to discuss with me in the comments below.


[echo-readme]:              https://github.com/ActiveState/stackato-echoservice#readme
[stackato]:                 http://www.activestate.com/stackato
[elasticsearch-service]:    https://github.com/ActiveState/stackato-elasticsearch-service/
[elasticsearch]:            http://www.elasticsearch.org/
[mongodb]:                  http://www.mongodb.org/
[mysql]:                    http://www.mysql.com/
[postgresql]:               http://www.postgresql.org/
[memcached]:                http://memcached.org/
[rabbitmq]:                 http://rabbitmq.com/
