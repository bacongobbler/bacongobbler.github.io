---
layout: post
title:  Deis on Digital Ocean
date:   2013-09-24
tags: deis devops paas howto
comments: true
---

Deploying applications to web servers on your own infrastructure these days has never been easier.

# Why the change?

When it comes to deployments, there has been a recent push for rapid release cycles, resource orchestration, application scalability, product delivery, and quality testing. IT Administrators and Software Developers need to be able to improve collaboration and communication. This mindset is often referred to as [DevOps][devops], a term used for integrating both software development and IT together.

DevOps can mean different things for different groups. Software Developers who really try to adopt a DevOps state of mind tend to stumble towards automated application deployment tools that provide application monitoring, scaling, and automated builds like Heroku, Windows Azure, or Google App Engine. IT Administrators who follow down the DevOps path want complete control over the hardware supporting their application. OpenStack, Rackspace, Amazon EC2, VSphere, and HP Cloud Services all come to mind. Clearly, both have two different optimal workflows which seem to be mutually exclusive. Heroku is awesome, and is hosted on Amazon EC2, but some companies have trust issues when it comes to offloading data on a public [PaaS][paas]. IT orchestration is cool and extremely helpful when deploying a new datacentre in a different region, but you miss out on the amazing features that a PaaS supplies for applications, which would include application scaling, monitoring, and the seemingly automagical deployment in the datacentre. But what if you could integrate the best of both worlds?

A private PaaS enables Devs and IT Admins to gain the benefits of the public cloud to deploy, manage, and monitor applications, while meeting the compliance issues that may come with that approach. AND, you get to keep working in the same workflow, making your developers super happy that they don't have to re-write their apps just to get them working!

Aside: I work at ActiveState Software Inc, where we have developed a private Platform as a Service called Stackato. Stackato is targeted towards large enterprise customers who want the flexibility of a Heroku-styled workflow while gaining control over an enterprise's servers. If you're looking for a PaaS where large companies like [Mozilla](http://insights.wired.com/profiles/blogs/upholding-the-open-web-with-paas-an-interview-with-mozilla-s-1#axzz2SGVdxYmm) or [Hewlett Packard](http://www.activestate.com/blog/2012/12/why-hp-chose-stackato-and-why-it-matters) are using in production, then feel free to drop me a call or an email!

# Introducing Deis

[Deis][deis] is an open source Application Platform that is meant for both public and private clouds. You get all the benefits of a Heroku-styled workflow for your users, all while gaining the flexibility of an IT automation tool.

The entire software stack is based on two core components: configuration management (for clustering) and containerization (for apps).

[deis]: http://deis.io/
[devops]: http://en.wikipedia.org/wiki/DevOps
[paas]: http://en.wikipedia.org/wiki/Platform_as_a_service
