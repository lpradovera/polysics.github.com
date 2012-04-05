---
layout: post
title: Plugins in Adhearsion 2.0 - Part 3
tags: [plugins, adhearsion]
last_updated: 2012-04-04
---

Adhearsion plugins provide many different mechanisms for extending and adding functionality to your application.

Following up [Part 1](http://mojolingo.com/blog/2012/plugins-in-adhearsion-2-0-part-1/) and [Part 2](http://mojolingo.com/blog/2012/plugins-in-adhearsion-2-0-part-2/),
this time we will be looking at adding custom generators to your plugin, and at how to use the #init and #run methods for initialization and dealing with execution of blocking
code such as an embedded server.

## Generators in an Adhearsion 2.0 plugin
Generators allow your plugin to create any type of file or structure in applications, based on templates and data.
Adhearsion generators are based on [Thor](https://github.com/wycats/thor), an excellent gem that provides a scripting framework, and the same library Rails generators are based on.

For our example, we will be adding a generator that creates a CallController with some default code and an include for the controller methods module our plugin provides,
which will already have been created.

### Generator class
The first necessary part for adding a generator is creating the main class as a subclass of Adhearsion::Generators::Generator, which itself inherits from Thor::Group.
We will be looking at a simple implementation, with additional options available in the Thor documentation.

The file will reside in a generators/ directory we add under the main plugin folder.


{% highlight ruby %}
# plugin-demo/lib/plugin-demo/generators/controller_generator.rb
module PluginDemo
  class ControllerGenerator < Adhearsion::Generators::Generator

    argument :controller_name, :type => :string

    def create_controller
      raise Exception, "Generator commands need to be run in an Adhearsion app directory" unless Adhearsion::ScriptAhnLoader.in_ahn_application?('.')
      self.destination_root = '.'
      template 'lib/controller.rb', "lib/#{@controller_name.underscore}.rb"
      template 'spec/controller_spec.rb', "spec/#{@controller_name.underscore}_spec.rb"
    end

  end
end
{% endhighlight %}

A generator class implements one or more public methods, "tasks" in Thor nomenclature, that will be executed when the class is run.
There is no naming scheme or specific convention, all tasks are un in the order they are defined in.

In this case we assume the generator will be invoked with a single argument that will be assigned to @controller_name.
In case of more than one argument, they will be assigned in succession.
The :type parameter allows for argument checking and implicitly converts it to the specified type.

After setting up, we simply check we are in an Adhearsion application, set the destination and copy over two files in the form of an ERb template.
Values are passed as instance variables set on the generator class.
Template content is as follows, with files placed in a templates/ directory and mimicking the structure they will have in the generated code.

{% highlight ruby %}
# plugin-demo/lib/plugin-demo/generators/controller/templates/lib/controller.rb
class <%= @controller_name %> < Adhearsion::CallController
  include PluginDemo::ControllerMethods # assume this module already exist
  def run
  end
end  

# plugin-demo/lib/plugin-demo/generators/controller/templates/spec/controller_spec.rb
describe <%= @controller_name %> do

end
{% endhighlight %}

The templates are very simple and follow ERb conventions.

### Registering and running our generator
After building the class and templates, the last step to is make Adhearsion aware of our generator by registering it.

{% highlight ruby %}
# plugin-demo/lib/plugin-demo/plugin.rb
require "plugin-demo/generators/controller_generator"

module PluginDemo
  class Plugin < Adhearsion::Plugin
    # rest of plugin code ...
    
    generators :"plugin_demo:controller" => PluginDemo::ControllerGenerator

  end
end
{% endhighlight %}

Registering our generator that way will make it invokeable through the CLI by using "ahn generate plugin_demo:controller MyController".

## Plugin initialization and running
A plugin will often want to go through some specific setup on initialization, and execute its own code or service when the Adhearsion application starts.
Additionally, it might depend on some other plugin being initialized or running.

Adhearsion provides two hooks that plugins can define to perform initialization and running steps, while requesting them to be executed before or after some other plugin.

Initialization is handled by the #init method, and its content will be executed when Adhearsion is loaded, before the application runs.
At this stage, we would be doing some "static" setup like ensuring presence of a resource or simply logging plugin loading.

When the application starts running, the #run method will be called. Here we would be running additional code, such as a service like a Sinatra web front-end or a DRb server.

{% highlight ruby %}
# example from the adhearsion-sinatra plugin, currently in development
module AdhearsionSinatra
  class Plugin < Adhearsion::Plugin
    # rest of plugin code ...
    
    init :sinatra, :after => :adhearsion_active_record do
      config = Adhearsion.config.sinatra
      @sinatra = Kernel.const_get(config['class_name']).new
    end
    
    run :sinatra do
      Thread.new { catching_standard_errors { @sinatra.run! } }
    end
  end
end
{% endhighlight %}

Both methods have a mandatory first parameter that is the name of the plugin to be used as a key in the initialization and running lists,
followed by optional :before and :after option keys that specify when to execute the hook. Code to run is passed as a block.

### Dealing with blocking execution
As you can see in the above example, services needing to run in their own thread or loop can be integrated in an Adhearsion plugin by wrapping the code that runs
them in a block passed to #catching_standard_errors. This allows the event system to catch, log and display in the console any exception of type StandardError raised by
the block.

You may or may not use a separate Thread for running services, though it is recommended.

## What's left?
Plugins can be very useful for your own development, but can also be shared with the community.
[AhnHub](http://ahnhub.com/) is a repository of Adhearsion plugins that allows developers to quickly find something they may need.

Contributing is simple: as shown on [the how-to page](http://ahnhub.com/how), you add a post-commit hook on your Github repository or a webhook on RubyGems and your plugin will
be automatically published to AhnHub. 
