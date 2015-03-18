---
layout: post
title:  "Stubbing library class methods in ChefSpec"
date:   2015-03-18
categories: chef chefspec chef-spec rspec ruby open-source opscode stubbing stub
---

One of the cool things about using Chef to automate your infrastructure is that
all of your infrastructure is represented as code.  An awesome benefit of having
that representation is the ability to make assertions and test expected outcomes
in by unit testing. [ChefSpec][chefspec], a unit testing library built on top of
Rspec, comes bundled in the [ChefDK][chefdk] and allows you to create unit tests for
cookbooks. Unit testing cookbooks allows you to quickly iterate during development
because your assertions are usually limited to verifying that the right resources
passed the right messages and took the proper actions.

In ChefSpec all of the chef-runs are all in memory and the actions are no-ops.
Execution time is very quick so the feedback loop is much better than waiting for
an integration test to fully run.

Often times when unit testing you'll find yourself stubbing various parts of your
program to isolate the pieces that you're actually trying to test. ChefSpec makes
it very easy to stub node attributes but stubbing class methods in library
Classes and Modules gets a bit tricky. Let's take a look at what I mean with an
example.

Here we have a cookbook called `demo` that has a default recipe and a library
that implements the Demo module. In the `Demo` module we're going to have a method
`foo()` that will return a string `bar`. The helper methods in your cookbooks are
probably going to be doing something more complicated but we'll run with this
example for simplicity.

{% highlight ruby linenos %}
# recipes/default.rb
log Demo.foo
{% endhighlight %} <p></p>

{% highlight ruby linenos %}
# libraries/demo.rb
module Demo
  def self.foo
    'bar'
  end
end
{% endhighlight %} <p></p>

Now write a ChefSpec test and attemp to stub `Demo.foo`

{% highlight ruby linenos %}
# spec/spec_helper.rb
require_relative '../libraries/demo'
require 'chefspec'
require 'chefspec/berkshelf'

describe 'demo::default' do
  let(:chef_run) { ChefSpec::SoloRunner.converge(described_recipe) }

  before { allow(Demo).to receive(:foo).and_return('override') }

  it 'stubs the demo library' do
    expect(chef_run).to write_log('override')
  end
end
{% endhighlight %} <p></p>

And run the test

{% highlight bash %}
ryan@/tmp/demo(master):> bundle exec rspec spec/spec_helper.rb
F

Failures:

  1) demo::default stubs the demo library
     Failure/Error: expect(chef_run).to write_log('override')
       expected "log[override]" with action :write to be in Chef run. Other log resources:

         log[bar]

     # ./spec/spec_helper.rb:11:in `block (2 levels) in <top (required)>'

Finished in 0.09651 seconds (files took 1.5 seconds to load)
1 example, 1 failure
{% endhighlight %} <p></p>

Ut oh, what's going on here? The answer lies deep in the heart of the `chef-client`.
When we call `.converge` on our ChefSpec runner we actually run through the entire
chef-client compile and execute phases. During that setup we have to compile our
cookbook which requires us to [load any library files][run_context] in the cookbook.

{% highlight ruby linenos %}
# lib/chef/run_context/cookbook_compiler.rb
def load_libraries_from_cookbook(cookbook_name)
  files_in_cookbook_by_segment(cookbook_name, :libraries).each do |filename|
    begin
      Chef::Log.debug("Loading cookbook #{cookbook_name}'s library file: #{filename}")
      Kernel.load(filename)
      @events.library_file_loaded(filename)
    rescue Exception => e
      @events.library_file_load_failed(filename, e)
      raise
    end
  end
end
{% endhighlight %} <p></p>

You'll see there on line #6 that we call `Kernel.load` on each library file in the
cookbook. Doing this will completely wipe out the stubs that we created on line #9
of our ChefSpec test.  So how do we fix this?

If your library is only going to be used during converge time you can stub it by passing
a block to `converge()` with the class stubs

{% highlight ruby linenos %}
# spec/spec_helper.rb
  let(:chef_run) do
    ChefSpec::SoloRunner.converge(described_recipe) do
      allow(Demo).to receive(:foo).and_return('override')
    end
  end

  it 'stubs the demo library' do
    expect(chef_run).to write_log('override')
  end
end
{% endhighlight %} <p></p>

That option won't work in our case because the library is being used at compile
time for the name of the log resource.

You could monkey patch the chef-client in a library and omit loading the library

{% highlight ruby linenos %}
# libraries/monkey_patches.rb
def load_libraries_from_cookbook(cookbook_name)
  files_in_cookbook_by_segment(cookbook_name, :libraries).each do |filename|
    begin
      Chef::Log.debug("Loading cookbook #{cookbook_name}'s library file: #{filename}")
      Kernel.load(filename)
      @events.library_file_loaded(filename) unless cookbook_name == 'demo'
    rescue Exception => e
      @events.library_file_load_failed(filename, e)
      raise
    end
  end
end
{% endhighlight %} <p></p>

That would work but it's a nasty hack at best.  The cleanest option I've found is to
write your library in a way that it won't be reloaded.

{% highlight ruby linenos %}
# libraries/demo.rb
module Demo
  def self.foo
    'bar'
  end
end unless defined?(Demo)
{% endhighlight %} <p></p>

When the cookbook compiler tries to load the Demo library again it won't
because we've previously loaded it in our `spec_helper.rb`

If we run our tests again with the modified library we should see it pass!

{% highlight bash %}
ryan@/tmp/demo(master):> bundle exec rspec spec/spec_helper.rb
.

Finished in 0.10684 seconds (files took 1.58 seconds to load)
1 example, 0 failures
{% endhighlight %} <p></p>

[chefspec]: https://github.com/sethvargo/chefspec
[chefdk]: https://downloads.chef.io/chef-dk/
[run_context]: https://github.com/chef/chef/blob/master/lib/chef/run_context/cookbook_compiler.rb#L187-198
