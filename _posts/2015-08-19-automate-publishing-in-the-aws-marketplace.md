---
layout: post
title:  "Automate publishing in the AWS Marketplace"
date:   2015-08-19
categories: chef ruby open-source opscode amazon aws marketplace ami ec2 cloud
---

At CHEF we recently started publishing the Chef Server along with it's Add-On's into
the [AWS Marketplace][marketplace]. The documentation
provided by Amazon was mostly limited to general guidelines and the rules of publishing.
While it recommends building an automated process for publishing, it stops short
of prescribing a repeatable way to build, test and publish AMI's into Marketplace.
What I've done is build a Chef resource that will allow you to easily automate
publishing your application into the AWS Marketplace. In this blog post I'll
give an overview of the entire process of creating a cookbook for your application
and detail how to use the `marketplace_ami` resource to automate the building and publishing
in Marketplace.

### Set up your workstation

Before we begin you'll need to install and configure the [Chef Development Kit][chefdk]
and the [AWS command line tools][awstools]. It's not required, but you'll probably
want to install [Virtualbox][virtualbox] and [Vagrant][vagrant] for local testing.
You can use Ec2 for testing as well.

### Prepare an application cookbook

Before you can automate the publishing of your application you first need to automate the
the installation and configuration of it.  In this section I'll go
through the process of writing a basic cookbook that will act as our app
in this tutorial.

If you're already familiar with and/or have a cookbook for automating your application
you should skip ahead to the [publishing section](#publish-your-application).

ChefDK comes bundled with several command line utilities to assist in building and
testing Chef cookbooks. Here we'll use the `chef` command to create a repository for your
chef policy and create a skeleton cookbook for the demo application
called `jackfruit` that we'll eventually publish to the AWS Marketplace.

* Create a chef-repo and application cookbook

{% highlight bash %}
$: cd ~
$: chef generate repo chef-repo
$: cd chef-repo/cookbooks
$: chef generate cookbook jackfruit
{% endhighlight %} <p></p>

For the sake of brevity we'll skip over the in's and out's of writing fully tested
idiomatic Chef cookbooks.  Instead, we'll limit the scope of this tutorial
to the task at hand.  There are lots of other great resources for [learning Chef][learnchef]
that I'd recommend going through if you're new to using it.

* Create a recipe that will set up Apache and serve our application

{% highlight ruby %}
# jackfruit/recipes/default.rb
include_recipe 'apt'

package 'apache2' do
  action :install
end

template '/var/www/htmp/index.html' do
  source 'index.html.erb'
  variables(
    app: 'jackfruit',
    message: node['jackfruit']['message']
  )
  notifies :restart, 'service[apache2]'
  action :create
end

service 'apache2' do
  action :start
end
{% endhighlight %} <p></p>

* Add 'message' to the node attributes

{% highlight ruby %}
# jackfruit/attributes/default.rb

default['jackfruit']['message'] = 'Welcome to the Jackfruit app on the AWS Marketplace!'

{% endhighlight %} <p></p>

* Create our index.html.erb template

{% highlight erb %}
# jackfruit/templates/default/index.html.erb
<html>
  <head>
    <title><%= @app %></title>
  </head>
  <body bgcolor=white>
    <h1>Welcome to <%= app %>!</h1>

    <p><%= @message %></p>
  </body>
</html>
{% endhighlight %} <p></p>

* Update the metadata to depend on the 'apt' recipe which ensures that our apt
mirrors are up to date when chef installs packages.

{% highlight ruby %}
# jackfruit/metadata.rb
name             'jackfruit'
maintainer       'ryan'
maintainer_email 'ryan@chef.io'
license          'ApacheV2'
description      'Installs/Configures jackfruit'
long_description 'Installs/Configures jackfruit'
version          '0.1.0'

depends 'apt'
{% endhighlight %} <p></p>

* Configure test-kitchen to forward port 80 from the VM to our workstation on 8080.

{% highlight yaml %}
# jackfruit/.kitchen.yml
---
driver:
  name: vagrant

provisioner:
  name: chef_zero

platforms:
  - name: ubuntu-14.04
    driver_plugin: vagrant
    driver_config:
      network:
        - ["forwarded_port", {guest: 80, host: 8080, auto_correct: true}]
  - name: centos-7.1

suites:
  - name: default
    run_list:
      - recipe[jackfruit::default]
    attributes:
{% endhighlight %} <p></p>

* Use test-kitchen to converge the VM and manually verify the output.  As you get more
familiar with Chef and test-kitchen you will want to automate this manual verification
using serverspec and test-kitchen suites.  If you opted to not install vagrant
and virtualbox you could use the AWS test-kitchen driver in this step.

{% highlight bash %}
$: cd ~/chef-repo/cookbooks/jackfruit
$: kitchen converge
{% endhighlight %} <p></p>

You should now be able to visit localhost in your browser and see our application.
Be careful to watch the test-kitchen output as it will automatically fix port
collisions, as it did in my case.

{% highlight bash %}
       ==> default: Fixed port collision for 80 => 8080. Now on port 2204.
{% endhighlight %} <p></p>

<img src="/images/jackfruit.png" align="middle">

Now that our demo application is automated we'll destroy the VM and move on.

{% highlight bash %}
$: kitchen destroy
{% endhighlight %} <p></p>

### Publish your application

For our publishing recipe we'll use the [marketplace_ami][marketplace_ami] resource
to provision a new EC2 instance, converge the jackfruit recipe, create and register
an AMI and share it with AWS Marketplace.

Optionally, you can enable a security recipe that will remove sensitive data by setting
`security` parameter to `true`. Also available is a chef-client audit mode
recipe that will audit the instance for known AWS security policies.  You can
enable it by setting the `audit` parameter to `true`.  Detailed information about
the resource can be found in the projects [README on github][marketplace_ami].

 * Update the jackfruit cookbook dependencies

{% highlight ruby %}
# jackfruit/metadata.rb
name             'jackfruit'
maintainer       'ryan'
version          '0.1.0'
...

depends 'apt'
depends 'marketplace_ami'
{% endhighlight %} <p></p>

* Create a publishing recipe

{% highlight ruby %}
# jackfruit/recipes/publisher.rb

marketplace_ami 'jackfruit-demo' do
  instance_type   't2.medium'
  source_image_id 'ami-123456'
  ssh_keyname     'your_ssh_key'
  ssh_keypath     '~/.aws/your_ssh_key.pem'
  ssh_username    'ec2-user'
  product_code    '123799879'
  security        false
  audit           false

  recipe 'jackfruit::default'
  attribute %w(jackfruit message), 'Awesome new content!'

  action :create
end
{% endhighlight %} <p></p>

In the example I used several but not of the `marketplace_image` resource's parameters.
Feel free to change the values to
reflect what you need.  If you omit parameters, the resource
and sub-resources will attempt to guess sane values but I'd recommend being as
explicit as possible to ensure a consistent and repeatable build.
In the example I've disabled the security and auditing checks because
our demo cookbook will fail the audit. This isn't a problem for the demo,
but when you're publishing your application you'll want to enable the audit during
build time and update your application cookbook until the audit passes.  This
will help to ensure that your image will pass the AWS security scanning.

* Create a Berksfile in your chef-repo that points to your application cookbook

{% highlight ruby %}
# ~/chef-repo/Berksfile
source 'https://supermarket.chef.io'

cookbook 'jackfruit', path: '~/chef-repo/cookbooks/jackfruit'

{% endhighlight %} <p></p>

* Now vendor all of the required cookbooks into your chef-repo

{% highlight bash %}
$: cd ~/chef-repo
$: berks vendor ~/chef-repo/cookbooks
{% endhighlight %} <p></p>

* Converge the the publisher recipe (output trimmed for clarity)

{% highlight bash %}
$: cd ~/chef-repo
$: chef-client -z -o jackfruit::publisher
Starting Chef Client, version 12.4.1
[2015-08-19T15:10:07-07:00] INFO:  Chef 12.4.1
[2015-08-19T15:10:09-07:00] INFO: Run List expands to [jackfruit::publisher]
resolving cookbooks for run list: ["jackfruit::publisher"]
- Create jackfruit-demo with AMI ami-0372b468 in us-east-1[2015-08-19T15:10:17-07:00] INFO: Processing chef_node[jackfruit-demo] action create (basic_chef_client::block line 57)
- jackfruit-demo is now ready[2015-08-19T15:10:50-07:00] INFO: [AWS EC2 200 0.215999 0 retries] describe_instances(:filters=>[{:name=>"instance-id",:values=>["i-0779e4ac"]}])
- jackfruit-demo is now connectable[2015-08-19T15:13:04-07:00] INFO: Reading key ryan from file /Users/ryan/.chef/keys/ryan
- generate private key (2048 bits)[2015-08-19T15:13:07-07:00] INFO: Executing sudo ls -d /etc/chef/client.pem on ec2-user@52.2.69.39
- write file /etc/chef/client.pem on jackfruit-demo[2015-08-19T15:13:08-07:00] INFO: Processing chef_client[jackfruit-demo] action create (basic_chef_client::block line 132)
- write file /etc/chef/ohai/hints/ec2.json on jackfruit-demo[2015-08-19T15:13:10-07:00] INFO: Executing sudo ls -d /etc/chef/client.rb on ec2-user@52.2.69.39
- write file /etc/chef/client.rb on jackfruit-demo[2015-08-19T15:13:11-07:00] INFO: Executing sudo chef-client -v on ec2-user@52.2.69.39
- run 'bash -c ' bash /tmp/chef-install.sh -v 12.4.1'' on jackfruit-demo[2015-08-19T15:22:21-07:00] INFO: Processing chef_node[jackfruit-demo] action create (basic_chef_client::block line 57)
-   update run_list from ["recipe[jackfruit::default]", "recipe[marketplace_ami::_securitycontrols]"] to ["recipe[jackfruit::default]", "recipe[jackfruit::default]", "recipe[marketplace_ami::_sec
[jackfruit-demo] [2015-08-19T22:22:24+00:00] INFO: Forking chef instance to converge...
                 Starting Chef Client, version 12.4.1
                 [2015-08-19T22:22:24+00:00] INFO: *** Chef 12.4.1 ***
                 [2015-08-19T22:22:26+00:00] INFO: Run List expands to [jackfruit::default, marketplace_ami::_security_controls]
                 resolving cookbooks for run list: ["jackfruit::default", "marketplace_ami::_security_controls"]
                 - chef-sugar
               Compiling Cookbooks...
               Converging 2 resources
               Recipe: jackfruit::default
               - Create image jackfruit-demo from machine jackfruit-demo with options {}[2015-08-19T15:38:43-07:00] INFO: [AWS EC2 200 0.154614 0 retries] describe_tags(:filters=>[{:name=>"resource-id",:values=>
               - applying tags {"From-Instance"=>"i-8b188520"}[2015-08-19T15:38:44-07:00] INFO: Processing chef_data_bag[machine_image] action create (basic_chef_client::block line 62)
               - create data bag machine_image at chefzero://localhost:8889[2015-08-19T15:38:44-07:00] INFO: Processing chef_data_bag_item[machine_image/jackfruit-demo] action create (basic_chef_client::block li
               - create data bag item jackfruit-demo at chefzero://localhost:8889
               - Image jackfruit-demo is now ready[2015-08-19T15:39:49-07:00] INFO: Processing aws_instance[jackfruit-demo] (i-8b188520) action destroy (basic_chef_client::block line 506)
               - delete instance aws_instance[jackfruit-demo] (i-8b188520) in VPC vpc-e42b2f81 in us-east-1[2015-08-19T15:39:50-07:00] INFO: [AWS EC2 200 0.212792 0 retries] describe_instances(:instance_ids=>["i
               - delete node jackfruit-demo at chefzero://localhost:8889[2015-08-19T15:40:01-07:00] INFO: Processing chef_client[jackfruit-demo] action delete (basic_chef_client::block line 44)
               - delete client jackfruit-demo at clients[2015-08-19T15:40:01-07:00] INFO: Processing chef_node[jackfruit-demo] action delete (basic_chef_client::block line 91)
             * ruby_block[share jackfruit-demo with the AWS Marketplace account] action run[2015-08-19T15:40:01-07:00] INFO: Processing ruby_block[share jackfruit-demo with the AWS Marketplace account] action ru
               - execute the ruby block share jackfruit-demo with the AWS Marketplace account
[2015-08-19T15:40:02-07:00] WARN: Skipping final node save because override_runlist was given
[2015-08-19T15:40:02-07:00] INFO: Chef Run complete in 594.834383 seconds
[2015-08-19T15:40:02-07:00] INFO: Skipping removal of unused files from the cache
{% endhighlight %} <p></p>

595 seconds to create, converge, audit, publish and share your AMI in the AWS Marketplace.
Not bad for 10 minutes of work.

### Run the security scanner

Login to the [AWS Marketplace Management UI][marketplace_manage] to locate your
new image.

<img src="/images/aws_marketplace.png" align="midddle">

Now you can intiate the AWS security scanning.  Eventually I hope to have this
step automated but there's not a public API and your Ec2/IAM credentials
cannot be used for authentication with the Marketplace Management console.

### Going further

In this example we used our workstation node to run the publishing recipe. Ideally
publishing would be done as part of an automated CI/CD pipeline job that you can
trigger.

We use this same process to publish our software into the marketplace.  You can find
our publishing recipes in our [marketplace_image cookbook][marketplace_image]

If you'd like to contribute, report bugs, or find more information, please visit
the [marketplace_ami github repo][marketplace_ami]

[marketplace]: https://aws.amazon.com/marketplace/seller-profile/ref=srh_res_product_vendor?ie=UTF8&id=e7b7691e-634a-4d35-b729-a8b576175e8c
[chefdk]: https://downloads.chef.io/chef-dk
[awstools]: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-set-up.html
[virtualbox]: https://www.virtualbox.org/wiki/Downloads
[vagrant]: https://www.vagrantup.com/downloads.html
[learnchef]: http://learn.chef.io/
[marketplace_ami]: https://github.com/chef-partners/marketplace_ami
[marketplace_image]: https://github.com/chef-partners/marketplace_image
[marketplace_manage]: https://aws.amazon.com/marketplace/management/manage-products/?#/manage-amis.shared
