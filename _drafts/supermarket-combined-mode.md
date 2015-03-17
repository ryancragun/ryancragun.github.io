---
layout: post
title:  "Supermarket combined mode"
date:   2015-03-12
categories: chef supermarket open-source opscode hosted-chef berkshelf
---

A couple of weeks ago I was chatting with a CHEF customer that uses the Hosted Chef
platform and they asked about the possibility of running their own private Supermarket.
The Supermarket makes cookbook collaboration between teams very easy
by providing a common repository for shared cookbooks and a Berkshelf API server
for handy cookbook resolution.  CHEF doesn't offer private Hosted Supermarket instances
but I didn't want them to miss out on the awesome that is the Supermarket. We
started looking into running the service in-house and interesting problems popped up.
Supermarket uses the [oc-id service][oc-id], an Oauth2 provider for Chef,
for identity and authorization.  Chef Server 11.2.0 and higher have this service
built-in but users of Hosted Chef don't have access to id.chef.io and running the
Supermarket without OAuth2 causes many functions to behave improperly.

Rather than running another OAuth2 service or building a bunch of hard to
maintain patches to the Supermarket we decided it would be easier to run
a dummy Chef Server to leverage oc-id and have Supermarket authenticate
with it.  If you try to install the Omnibus Supermarket and a Chef Server in their default modes
on the same server you're going to have a pretty [bad time][badtime].  The Chef Server
and Supermarket are both web applications that share a lot of similar services
that expect to bind to common ports and don't play nicely together.
It might have been possible to get both running by swapping ports and other
tomfoolery but you'd still have duplicate services and a snowflake
installation.  That was not a yak I was willing to shave. Another option
that takes considerably less effort would be running each service on it's own
Server which seems like a lot of wasted overhead for a lightly used
Oauth2 service.  Enter Supermarket 'combined' mode.

Supermarket combined mode piggy backs onto a Standalone Chef Server and uses its
nginx, Postgresql and redis services, leaving only the Supermarket Rails application and
Sidekiq workers as the only new services.  Along with this new mode I also migrated
the Supermarket config file to [Mixlib::Config][mixlib-config] to make configuring
`/etc/supermarket/supermarket.rb` consistent with all of the other CHEF software
software that you're used to running.  Sorry [Nathan!][nlsmith]

Configuring combined mode is super simple.  First, [download][packagecloud] and
install the latest Chef Server and Supermarket Omnibus packages.

{% highlight bash %}
$: dpkg -i chef-server-core.deb supermarket.deb
{% endhighlight %} <p></p>

Edit the configuration file and set the topology to `'combined'`

{% highlight ruby %}
#/etc/supermarket/supermarket.rb

topology 'combined'
{% endhighlight %} <p></p>

And reconfigure Chef Server and Supermarket

{% highlight bash %}
$: chef-server-ctl reconfigure
$: supermarket-ctl reconfigure
{% endhighlight %} <p></p>

Bingo, now you've got a fully functional Supermarket! If you navigate
to the Servers FQDN in your browser you should the Supermarket running in all of
its beautiful glory.

<img src="/images/supermarket_ui.png">

But what good is a supermarket if you can't authenticate and upload cookbooks?
Lets create some users for each person that's allowed to publish to the Supermarket.
If you're running Chef Server 12 (which you are because you're cool and so is
Chef Server 12) then you can do this easily using the shell.

{% highlight bash %}
$: chef-server-ctl user-create username firstname lastname username@company.org password
{% endhighlight %} <p></p>

This will spit out a private key that you'll want to keep around if you plan to
publish to the Supermarket using [Stove][stove] or the like. If you're running
Enterprise Chef 11.x you can install [knife-opc][opc] and create users with knife.

Now that we've got a user navigate back to the Supermarket in your browser and login.  After
you login you'll be prompted to authorize the Supermarket.

<img src="/images/oc_id_auth.png">

Agree and you're all set.

[oc-id]: https://github.com/chef/oc-id/
[mixlib-config]: https://github.com/chef/mixlib-config
[nlsmith]: https://github.com/chef/omnibus-supermarket/pull/13#discussion_r26095884
[stove]: https://github.com/sethvargo/stove
[opc]: https://github.com/chef/knife-opc
[badtime]: http://southparkstudios.mtvnimages.com/images/shows/south-park/clip-thumbnails/season-6/0603/south-park-s06e03c03-thumper-the-super-cool-ski-instructor-16x9.jpg
[packagecloud]: https://packagecloud.io/chef/stable
