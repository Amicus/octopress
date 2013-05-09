---
layout: post
title: "ClusterFsck: Filebased S3 Configuration Kontrol for software clusters"
author: Brian Glusman
picture: "/images/brian8bit.png"
date: 2013-05-09 09:46
comments: true
categories:
---

Configuration may not be the sexiest problem to deal with, but it can ruin an otherwise well engineered piece of software when it's done poorly. Configuration problems can complicate or delay software updates, changes, deployments and error replication when it's hard to see and change centrally or hard to replicate and control in different environments.

ClusterFsck is our solution to configuration challenges at Amicus, and we want to share. If you want to follow along on your own project, you can `gem install clusterfsck` and proceed through the equivalent steps for your application.

<!--more-->

Centralizing your configuration can help a number of ways, from giving you access control levels for free, making replicating production problems locally easier, and changing configuration at runtime without having to redeploy, or even without restarting the server.  There are still some security issues to be conscious of when using ClusterFsck, and Jonathan Simms (@slyphon) pointed out some we hadn't thought of when taking a look at ClusterFsck for us:

* Don't ever ever ever put sensitive credentials in environment variables if there's another option.  On Heroku, this may still be your best option, but on other platforms there are better choices, because `ps` can show the environment of a running process.

* Be careful about the permissions on all configuration files with sensitive data, including ClusterFsck's config files. We plan to add automatic checks for this to ClusterFsck and show a warning if they're group/world writable or readable, though a pull request doing this for us would be welcome ;-)

But assuming you're sold on the benefits and know the issues to watch out for, you can jump straight to [ClusterFsck's github page](https://github.com/amicus/clusterfsck) or follow along here as we set up a simple usage of it.

Let’s say you want to store an API key for Amicus in your project and it needs to be different in development/staging/production.

### Step 1 - Create the configs
```bash
clusterfsck create development amicus-api
```

By default cluster-fsck will look at environment variables, `./.clusterfsck`,  `/usr/clusterfsck`, `~/.clusterfsck` (see [readme.md](https://github.com/Amicus/clusterfsck/blob/master/README.md)).  It looks for S3 credentials in its own config file, or in a `~/.fog file`.  Let’s assume it finds your `~/.fog` keys.

```bash
$ > clusterfsck create amicus-api
```

ClusterFsck stores your configuration(s) in an S3 bucket, which must have a unique (global) name.
The name will be stored in `~/.clusterfsck` on this machine, and should be placed in
`/usr/clusterfsck` on your production box (with a different CLUSTER_FSCK_ENV setting if desired).

```bash
Enter a name for your bucket, or press enter to accept the randomly generated name:
clusterfsck_too_bad_impiety
bucket name:
```

If you accept the random bucket name by pressing enter, you’ll see:
```bash
WARNING: clusterfsck_too_bad_impiety does not exist.  Type yes below to have it created for you, or return to abort.
Create bucket clusterfsck_too_bad_impiety?:
```
Type ‘yes’ and the bucket is created.  The bucket name is stored in `~/.clusterfsck` and the environment is set there as development.

It will then open your `$EDITOR` with an empty yaml file (the equivalent of YAML.dump({})).  This is your YAML config for the development environment.

```yaml
--- {}
```
Drop in your API key...
```yaml
---
key: abc123
secret: def446
```
You can go ahead and create separate keys for your staging and production environments.
```bash
clusterfsck create staging amicus-api
```

### Step 2 - Use the config

See the [readme.md](https://github.com/Amicus/clusterfsck/blob/master/README.md) for setting up the config and CLUSTER_FSCK_ENV.  CLUSTER_FSCK_ENV defaults to ‘development’
```ruby
ClusterFsck.environment = “staging”
```
In your app you can now
```ruby
reader = ClusterFsck::Reader.new(“amicus-api”).read
# which returns Hashie::Mash.new({key: “abc123”, secret: “def446”})
# which you can use as such:
reader.key == ‘abc123’
#or
reader[:key] == ‘abc123’
```

That’s it! Hopefully you can see how to proceed from here, and know which projects might benefit from a little bit of cluster-unfscking and configuration extraction. Feel free to let us know if you have any ideas or requests for future improvements to ClusterFsck!

