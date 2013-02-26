---
layout: post
title: "How We Provision and Deploy (Part 1) - Provisioning"
author: Topper Bowers
picture: /images/topper.png
date: 2013-02-26 13:20
comments: true
categories: 
---

Here at Amicus, we have a fairly novel stack for our provisioning and deployment system. We are currently at a happy spot, so we thought we’d share what we’re doing with the world.

A bit of background (that’ll I’ll cover more and more of in future blog posts).

* We are a jruby shop (1.7.x) but mostly develop against MRI (1.9.3)
* We run Rails and some limited Sinatra
* We provision and deploy on EC2
* We use Mongo as our primary data store with some Redis and Zookeeper in the mix (the latter two are new)
* Our philosophy is “always be horizontally scalable” and “no single points of failure.”
* We haven’t gone down during the recent AWS outages (even though we are in the east region)

<h2>Chef</h2>

All of our infrastructure begins with <a href="http://wiki.opscode.com/display/chef/Chef+Solo">chef-solo</a> and <a href="https://github.com/fog/fog">Fog</a>.

We elected to not setup chef-server.  While chef-server does provide a lot of niceities, the people we know who are running it themselves seem to have a lot of headaches.  From compacting the couchdb, to having to handle backup and uptime on an EC2 instance - there is a fair amount of work that goes into maintaining a chef-server.  It’s also a SPOF and as you know from above, we don’t do that.

We, instead, use the AWS API to assign tags to the instances we spin up.  A script that runs before chef-solo will look at those tags and decide what needs to be run from chef.

Let’s spin up a server (on my local box).

<code>
bin/create_server -e staging -g staging -r webapps -n webapps00
</code>

That will use Fog to go out to AWS and create a server in the staging security group (<code>-g</code>) and staging environment (<code>-e</code>, i’ll get to this in a sec).  Adds a Name tag (<code>-n</code>) of <code>webapps00</code> and a “roles” (<code>-r</code>) tag of “webapps.”

We use a bare Ubuntu 12.04 AMI for this (always starting from scratch) and our user-data script installs ruby, chef-solo and clones our deployer code (the same project from which I ran create_server on my local box).

In addition, our <a href="http://alestic.com/2009/06/ec2-user-data-scripts">user-data</a> script uses the built-in http metadata lookups available to every <a href="http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/AESDG-chapter-instancedata.html">EC2 instance</a> to determine what roles need to be run on the machine.  Because the machine exists in the ‘staging’ environment (basically, environment is only a tag on the box).  The machine is also able to use the AWS API to figure out what other machines are in this environment.

You can think of the environment tag as simulated multicast... when a webapps box spins up, it is able to determine the addresses of other necessary boxes (like the mongo cluster) using discovery rather than coded values.

The user-data script is able to spit out a JSON file that describes the roles for this machine and ALSO all the other machines in the ‘staging’ environment.  Our chef recipes are then able to do something like:

<code>
@mongo_nodes = node[:environment][:mongo].map(&:private_ip)
</code>

This lets us spit out configs dynamically without having a configuration server or a chef-server.

We have made a conscience decision to only use instance store (even for Mongo) and that decision has served us well through the AWS outages as those have mostly been EBS related.  

One thing we learned from the Obama For America organization is that it might be interesting to use a mix of instance-store and EBS-backed instances.  Instance-store for redundancy and EBS-backed for spin-up speed.  Currently from typing create server on the command line to a running server is about 4-8 minutes for us (depending on roles).  That’s probably too long for auto-scaling.  However, it’s only on the border of too-long and adding an EBS snapshot starting point should be able to keep this exact system but get the time down drastically.

To sum it up:

* chef-solo not chef-server
* use the AWS tags for server identification
* nodes know about the other nodes in the environment
* instance store not EBS

Next Up: App Deployment