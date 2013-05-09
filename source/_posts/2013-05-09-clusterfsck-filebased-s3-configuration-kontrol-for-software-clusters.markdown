---
layout: post
title: "ClusterFsck: Filebased S3 Configuration Kontrol for software clusters"
author: Brian Glusman
picture: "/images/brian8bit.png"
date: 2013-05-09 09:46
comments: true
categories:
---

Configuration may not be the sexiest problem to deal with, but it can ruin an otherwise well-engineered piece of software when it's done poorly. Configuration problems can complicate or delay software updates, changes, deployments and error replication.

[ClusterFsck](https://github.com/amicus/clusterfsck) is our solution to central configuration management at Amicus, and we want to share.

<!--more-->

ClusterFsck provides a clean and easy way to store configurations on S3. With ClusterFsck, you edit config files locally and easily read them in your app code.  ClusterFsck is environment aware, allowing you to centrally maintain (for instance) staging, production, dev configs.

Centralizing your configuration helps in a number of ways:
  * ACL for free (because it's on S3)
  * Keep your configs out of your app repos (sekuritah!)
  * Change configs without a redeploy
  * Keep development configs consistent across the team

Jump straight to [ClusterFsck's github page](https://github.com/amicus/clusterfsck) or keep reading as we set up a simple usage.

Let’s say you want to store an API key for Amicus in your project and it needs to be different in development/staging.

### Step 1 - Create the configs
```bash
clusterfsck new amicus-api
```

ClusterFsck stores amicus-api in your configured S3 bucket.

It will then open your `$EDITOR` with an empty yaml file. This is your YAML amicus-api config for the development environment (development is default).

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
clusterfsck new staging amicus-api
```

### Step 2 - Use the config

See the [readme.md](https://github.com/Amicus/clusterfsck/blob/master/README.md) for setting up the config and CLUSTER_FSCK_ENV.
```ruby
ClusterFsck.environment = “staging”
```
In your app you can now
```ruby
config = ClusterFsck::Reader.new(“amicus-api”).read
# which returns Hashie::Mash.new({key: “abc123”, secret: “def446”})
# which you can use as such:
config.key == ‘abc123’
#or
config[:key] == ‘abc123’
```

That’s it! I'm sure you have projects that will benefit from a little bit of cluster-unfscking and configuration extraction. Let us know if you have any ideas or requests for future improvements to ClusterFsck!

