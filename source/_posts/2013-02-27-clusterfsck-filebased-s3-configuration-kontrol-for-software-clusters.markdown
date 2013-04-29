---
layout: post
title: "ClusterFsck: Filebased S3 Configuration Kontrol for software clusters"
author: Brian Glusman
date: 2013-02-27 09:46
comments: true
categories:
---

Configuration may not be the sexiest problem to deal with or think about, but it can ruin an otherwise well engineered piece of software when it's done badly, and can complicate or delay software updates, changes, deployments and error replication when it's hard to see and change centrally or hard to replicate and control in different environments.

<!--more-->

ClusterFsck is our solution to virtually all configuration challenges at Amicus, and we wanted to share it with the world and invite other developers to leave configuration nightmares behind them.  All you need is an AWS account and a bucket key, the cluster-fsck ruby gem, and this article to walk you through centralizing your configuration to a few YAML files.

In a future post, I plan to discuss our decision to run a private rubygems.org server for some of our other internal gems and the process of adapting the code base for our use, but for now we'll use that project as a good example of adapting an existing project's configuration to use ClusterFsck.

If you want to follow along on your own project, you can `gem install cluster-fsck` and proceed through the equivalent steps for your application.

The first time you run ClusterFsck through it's CLI, it will pull its configuration from one of a few locations, or prompt you to accept a generated bucket name and to provide S3 keys, and then store a configuration for you in `~/.clusterfsck`.  The other locations it checks for it's config are `/usr/clusterfsck` and in the local directory where it was run from, `./.clusterfsck`, checking usr, then home, then local.  It also looks for S3 keys in a `~/.fog` file and for any or all of it's config keys in environment variables.  It will also check if the bucket exists and offer to create it if it does not.

Having gone through RubyGems.org and identified some pieces of configuration we'd like to pull out, we name each piece and put it in a YAML file under a project identifier in ClusterFsck's bucket...  ClusterFsck can create and edit these files your you, or you can edit them yourself, but for now let's use the `clusterfsck create rubygems` command, which will create a rubygems YAML file inside the default environment folder of your ClusterFsck bucket (`development` by default), and then bring up your default editor to edit that file.  The contents of the file must be valid YAML, or ClusterFsck will raise an error trying to save it.  The default file will look like this 
`--- {}` and simply using individual key/value pairs is the easiest way to pull out configuration to ClusterFsck.  After adding some of our RubyGems config to our file, it looks like this:

```yaml
  HOST: private-amicus-rubygems.herokuapp.com
  S3_KEY: AKIA************7DHA
  S3_SECRET: Vm+*********************************/yDF
  AUTH_PASSWORD: supersecretpasswordexample
  AUTH_REALM: amicus-rubygems
  REDIS_URL: redis://redistogo:ecredisbasicauthpassword@dory.redistogo.com:9958/
  S3_BUCKET: possibly-secret-s3-bucket-name
  S3_DOMAIN: some.aws.domain.or.custom.domain.com
  CF_DOMAIN: another-domain-for-cloud-front.com
```

We haven't touched the project yet, just pulled out the information that it needs that we'd rather wasn't hard coded into the repository or the local environment.  Next we have to replace the references to this data with requests to ClusterFsck to get the data.

For now, we're going to do this with a nice ugly global variable to store our ClusterFsck Reader object, after we've added the ClusterFsck gem to our project;  we'll put it in `config/application.rb` since that's one of the main places we'll use it, but this way we can get to it elsewhere without having to initialize it each time:
```ruby
$reader          = ClusterFuck::Reader.new(:rubygems)
```

One of the places we'll use it is in RubyGems `config/initializers/s3.rb` file.  The original file looked like this:
```ruby
if ENV['S3_KEY'] && ENV['S3_SECRET']
  Fog.mock! if Rails.env.test?

  $fog = Fog::Storage.new(
    :provider               => 'AWS',
    :aws_access_key_id      => ENV['S3_KEY'],
    :aws_secret_access_key  => ENV['S3_SECRET']
  )
end
```
We can replace this with:

```ruby
if $reader.read['S3_KEY'] && $reader.read['S3_SECRET']
  Fog.mock! if Rails.env.test?

  $fog = Fog::Storage.new(
    :provider               => 'AWS',
    :aws_access_key_id      => $reader.read['S3_KEY'],
    :aws_secret_access_key  => $reader.read['S3_SECRET']
  )
end
```

Now, along with the keys we already created in our ClusterFsck bucket above, that part of our ClusterFsck conversion is done!  The rest of the project, and of your own project, should be similarly easy, though you can also store entire arrays or hashes of config data in a single key, to handle options or a changing group of exceptions or special cases.  You can also use ClusterFsck to replace all or parts of YAML files that your application uses for config.  In RubyGems, there is a rubygems.yml file relied on by several parts of the application, though only a few parts were likely to change for us.  Some YAML files may already be processed by ERB, allowing you to interpolate ClusterFsck values into them as the output from ERB interpolation, but if not, it's easy to modify the YAML file loading to pass through ERB, as we did in the RubyGems `config/application.rb`.

We simply replaced this: <br />
`$rubygems_config = YAML.load_file("config/rubygems.yml")[Rails.env].symbolize_keys` <br />
with this:<br />
`$rubygems_config = YAML.load(ERB.new(IO.read('config/rubygems.yml')).result)[Rails.env].symbolize_keys`

You could create your own YAML ERB loader to do this more cleanly of course, or even monkey-patch Yaml.load_file, but that's up to you.  Similarly, the whole file could be output from ClusterFsck, but those are details you can decide for yourself.  Hopefully you can see how to proceed from here, and know which projects might benefit from a little bit of cluster-unfscking and configuration extraction, and feel free to let us know if you have any ideas or requests for future improvements to ClusterFsck!