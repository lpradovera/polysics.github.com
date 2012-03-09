---
layout: post
title: Plugins in Adhearsion 2.0 - Part 2
tags: [plugins, adhearsion]
last_updated: 2012-03-08
---

In our exploration of a newly generated plugin, we have so far mostly looked at the facilities Adhearsion provides to hook into the framework and your application.
It is now time to actually build some new business logic, although it is entirely possible to have a plugin that consists of Rake tasks or configuration variables only.

### lib/controller_methods.rb

A plugin is at its heart simply a Ruby gem, and bundled code needs to be loaded through requiring the proper files.

The generated plugin has a single business logic file in lib/controller_methods.rb.
Neither the file name nor the module name are mandatory, this is just normal Ruby code.

#### Generated code
Content is as follows:
{% highlight ruby %}
# lib/controller_methods.rb
module GreetPlugin
  module ControllerMethods
    # The methods are defined in a normal method the user will then mix in their CallControllers
    # The following also contains an example of configuration usage
    #
    def greet(name)
      play "#{Adhearsion.config[:greet_plugin].greeting}, #{name}"
    end
  end
end
{% endhighlight %}
The module is intended to be used as a mixin in call controllers.


#### Testing your code
Module usage can be seen in action in the generated test file, which also showcases how the call controller methods can be easily tested.
{% highlight ruby %}
# spec/greet_plugin/controller_methods_spec.rb
require 'spec_helper'

module GreetPlugin
  describe Plugin do
    describe "mixed in to a CallController" do
      
      class TestController < Adhearsion::CallController
        include GreetPlugin::ControllerMethods
      end

      let(:mock_call) { mock 'Call' }
    
      subject do
        TestController.new mock_call
      end

      describe "#greet" do
        it "greets with the correct parameter" do
          subject.expects(:play).once.with("Hello, Luca")
          subject.greet "Luca"
        end
      end
    
    end
  end
end
{% endhighlight %}
Since plugin code is a normal Ruby library, it can be tested in an easy way using Rspec and some facilities provided by Adhearsion.

## Using the plugin
You have generated your new plugin, built tests and are ready to use it. Now what?

The plugin is a gem, so you might eventually publish it, but you can simply use it locally by adding a path line to your application's Gemfile.

{% highlight ruby %}
# your application's Gemfile
gem 'greet_plugin', :path => '/home/user/projects/greet_plugin'
# ... rest of Gemfile
{% endhighlight %}

Do not forget to run 'bundle install' to load eventual dependencies after adding the gem.

## Adding an entire CallController
It is also possible to provide a full CallController implementation that can be used out-of-the-box by your application.
Ben Langfeld's excellent [blog post](http://mojolingo.com/blog/2012/adhearsion-2-call-controllers-routing/) explains how a CallController works and what you can do with it.

We will be adding a new controller to our plugin, complete with new configuration keys and test.
Our goal is to have a controller that dials a SIP device during office hours, and plays a message when the office is closed.

### Setup

First, we add configuration for the times used:
{% highlight ruby %}
# lib/greet_plugin/plugin.rb
config :greet_plugin do
  greeting "Hello", :desc => "What to use to greet users"
  office_hours_start 8, :desc => "Start of office hours, integer, 24 hour format"
  office_hours_end 18, :desc => "End of office hours, integer, 24 hour format"
end
{% endhighlight %}

To ease testing of time-based features, we add the excellent [Timecop](https://github.com/jtrupiano/timecop) gem to our gemspec, under the development group, and add "require 'timecop'" at the top of spec/spec_helper.rb.

### Tests first!

We then add a few tests, taking advantage of Adhearsion testing facility and the generated helpers. The tests describe what we will be implementing in the controller.
{% highlight ruby %}
#spec/hours_controller_spec.rb
require 'spec_helper'

module GreetPlugin
  describe HoursController  do
      let(:mock_call) { mock 'Call' }

      subject do
        HoursController.new mock_call
      end

      it 'dials out when inside office hours' do
        Timecop.freeze(Time.utc(2012, 3, 8, 12, 0, 0))
        subject.expects(:dial).once
        subject.run
      end

      it 'plays a message when outside office hours' do
        Timecop.freeze(Time.utc(2012, 3, 8, 22, 0, 0))
        subject.expects(:play).once
        subject.run
        
      end

  end
end
{% endhighlight %}

### Our CallController

Last but not least, we build the actual CallController.
{% highlight ruby %}
# lib/greet_plugin/hours_controller.rb
module GreetPlugin
  class HoursController < Adhearsion::CallController

    def run
      if Time.now.hour.between?(Adhearsion.config[:greet_plugin].office_hours_start, Adhearsion.config[:greet_plugin].office_hours_end)
        dial "SIP/101"
      else
        play "Office is  open between #{Adhearsion.config[:greet_plugin].office_hours_start} and #{Adhearsion.config[:greet_plugin].office_hours_end}."
      end
    end

  end
end
{% endhighlight %}

## Routes

To allow our application to reach the controller, we add routes in its configuration.
For the purpose of this post, we will simply be using the default route.
{% highlight ruby %}
#my_application_dir/config/adhearsion.rb
Adhearsion.router do
  route 'default', GreetPlugin::HoursController
end
{% endhighlight %}

## Conclusion
In this post we have highlighted how easy it is to add reusable and tested functionality to your Adhearsion applications using plugins.
The next post will deal with adding custom generators to your plugin.
