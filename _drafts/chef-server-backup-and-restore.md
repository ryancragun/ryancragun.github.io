---
layout: post
title:  "Introducting chef-server-ctl backup and restore"
date:   2015-03-19
categories: chef chef-server open-source opscode backup restore
---

Several months ago I started working on a little project called [chef_backup][:chef_backup]
that does exactly what it sounds like: backs up and restores the Chef Server.
The project sat partially implemented in my backlog for a long time, but as demand
for a first class backup solution for the Chef Server increased among the CE team
at CHEF, I decided that rather than wait until I'd written all of the backup strategies
that it'd be better to ship something that works and add the other functionality later.

I'm happy to announce that as of Chef Server %%% VERSION %%% the first version of
feature has been merged and released!  Support is pretty limited but future upgrades
are planned and should ship as time allows.  The majority of the heavy lifting lives in
the gem so it'll be pretty easy to upgrade the library in place on your Chef Server.
As we strive to continually deliver the Chef Server quickly I expect the updates
to ship bundled in the Chef Server frequently.

## chef-server-ctl backup
The backup functionality is controlled by a strategy class implementing the a
`backup()` method.  Currently there's only one strategy, 'tar', that will temporarily
stop the Chef Server, backup the Chef Server and Chef Server add-ons configuration
directories, Chef Server variable state, and upgrade data.  By default it will
also do a full pg_dump, but since we backup the raw postgresql data you don't
necessarily need that.  You can opt to disable it if you'd like the backups to
go faster.  In the future I hope to support online backups that utilize LVM for
people that are using the HA topology.  EBS snapshot and Object based
[knife ec backup][:ecbackup] support is on the backlog too.

### Usage

Using the tool is super simple.  Just set the backup options
in `/etc/opscode/chef-server.rb`, reconfigure and run the backup!

{% highlight bash %}
chef-server-ctl backup
#=> chef-backup-2015-03-19-23-52-32.tgz
{% endhighlight %} <p></p>

### Configuration

{% highlight ruby %}
# /etc/opscode/chef-server.rb

# Configure the backup strategy.
# The only supported strategy right now is 'tar'
backup['strategy'] = 'tar'

# Configure the directory you want to ship backups to
# defaults to /var/opt/chef-backups
backup['export_dir'] = '/mnt/mynas/backups'

# Running the backup means writing out some files.  Make sure directory has
# enough disk space.  This will be more important for the knife ec backup
# strategy.
# defaults to a unique directory in /tmp
backup['tmp_dir'] = '/temporary/directory/to/use/during/backup'

# Configure whether or not you want to include a database dump.
# defaults to true
backup['always_dump_db'] = true

# Configure whether or not you want the Chef Server to be online or offline
# during the backup.
# 'online' is super dangerous in 'tar' mode and should only be used if you don't
# care about consistency.
# defaults to 'offline' in 'tar' strategy
backup['mode'] = 'offline'

# Do you want to log the output?  Cool, tell it where.  This isn't enabled by
# default but can be useful if you're backing up your chef-server via cron
backup['logfile'] = '/path/to/log.txt'
{% endhighlight %} <p></p>

### chef-server-ctl restore

Restoring the Chef Server is as simple as installing the `chef-server-core` package
and running the restore command with your backup tarball.

{% highlight bash %}
chef-server-ctl restore backup.tgz [options]
{% endhighlight %} <p></p>

Optionally, you can pass a few parameters.

* -d, --staging-dir

  A directory to use for expanding the tarball.  If you have a large backup make
  sure the directory is large enough for the expanded contents.  If you don't
  pass anything it will create it's own directory in /tmp


* -c, --cleanse

  This will bypass the 60 seconds we wait when cleasing existing state of the
  Chef Server.

## More strategies

Right now I'm working on getting the LVM strategy working.  I'm really excited to
get support done because having to bring the Chef Server offline for a backup
isn't all that delightful.  After that we'll try and tackle Object and EBS backups.

Please give the new functionality a try and provide feedback on how we can make it
better.

[:chef_backup]: https://github.com/ryancragun/chef_backup
[:ecbackup]: https://github.com/chef/knife-ec-backup
