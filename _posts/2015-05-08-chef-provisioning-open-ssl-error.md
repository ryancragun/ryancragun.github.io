---
layout: post
title:  "Fixing the strange chef.io SSL errors when using Chef Provisioning"
date:   2015-05-08
categories: chef chef-server open-source ruby ssl openssl opscode chefdk provisioning
---

TLDR: Install the [ChefDK][:chefdk] for great good.

Today I was scheduled to work with another developer on Chef Provisioning on AWS
so I decided to dust off a few Chef Provisioning recipes I had lying around from
a demo a few months back.  When I tried to provision one of them I was greeted with
a rather alarming problem

{% highlight bash %}
ERROR: SSL Validation failure connecting to host: www.chef.io - SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed
{% endhighlight %} <p></p>

Interesting.  Why are we contacting chef.io and why can't I validate the SSL
certificate?  After running `chef-client` with debug logging I determined that
Chef Provisioning was attempting to download and cache the chef-client omnibus
package for the machines that I was provisioning.  The second issue is a much
bigger problem, but before we go any further I have to mention that this wouldn't
have been a problem at all if my `chef-client` had been installed using the
[ChefDK][:chefdk].
As as Ruby developer, I find myself using chruby to bounce between versions and
so I was using Chef the old-school manual gem install way.  Big mistake.

Rather than switch to the ChefDK and getting back to productive work, I decided to
hunt down the root cause.  After verifying that the latest OpenSSL version was
installed and linked, my system gems were updated, and that I'd installed the
latest security bundle from Apple, I figured it must be a ca-cacert issue.

First I determined which cert Ruby was using

{% highlight bash %}
$: ruby -r openssl -e "p OpenSSL::X509::DEFAULT_CERT_FILE"
"/usr/local/etc/openssl/cert.pem"
{% endhighlight %} <p></p>

Then updated updated it to reflect Apple's latest certs

{% highlight bash %}
$: security find-certificate -a -p /Library/Keychains/System.keychain > /usr/local/etc/openssl/cert.pem
$: security find-certificate -a -p /System/Library/Keychains/SystemRootCertificates.keychain >> /usr/local/etc/openssl/cert.pem
{% endhighlight %} <p></p>

Then tried running the provisioner again

{% highlight bash %}
$: chef-client -z provisioner.rb
...
ERROR: SSL Validation failure connecting to host: www.chef.io - SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed
{% endhighlight %} <p></p>

Damn.  Okay, maybe Apple's certs are no good.  Let's try the latest bundle from the
curl/Mozilla folks.

{% highlight bash %}
$: curl http://curl.haxx.se/ca/cacert.pem -o /usr/local/etc/openssl/cert.pem
$: chef-client -z provisioner.rb
...
ERROR: SSL Validation failure connecting to host: opscode-omnibus-packages.s3.amazonaws.com - SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificat
e verify failed
{% endhighlight %} <p></p>

Sweet, certificate validation worked on chef.io but now we're unable to verify the
S3 cert.  We're stuck in limbo here because Apple's bundle doesn't like
chef.io/fastly's cert and curl's bundle doesn't like S3.  After trying to verify
the S3 cert I stumbled upon a [forum post][:s3forum] by Lamont that details that
support for 1024 bit signing keys has been phased out.  Yikes.  Until AWS reissues
their cert with a proper root signature we either have to abandon S3 or use an
older certificate bundle, the latter of which is how we've solved that problem
in the ChefDK.

I snagged a [copy of the certificate bundle from last August][:cabundle] (before
1024 bit keys were phased out) and gave it a go

{% highlight bash %}
$: curl https://s3.amazonaws.com/uploads.hipchat.com/7557/2027616/rPfbosc2b4pPNh8/cacert.pem -o /usr/local/etc/openssl/cert.pem
$: chef-client -z provisioner.rb
...
- create new file /Users/ryan/.chef/package_cache/chef_12.3.0-1_amd64.deb
...
{% endhighlight %} <p></p>

Viola!  Using the old bundle obviously isn't ideal, but it unblocked me this time.

[:cabundle]: https://s3.amazonaws.com/uploads.hipchat.com/7557/2027616/rPfbosc2b4pPNh8/cacert.pem
[:chefdk]: https://downloads.chef.io/chef-dk
[:s3forum]: https://forums.aws.amazon.com/thread.jspa?threadID=164095
