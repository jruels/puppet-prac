# Write a module 

## Editing a Forge module
Although many Forge modules are exact solutions that fit your site, many are almost but not quite what you need. Sometimes you will need to edit some of your Forge modules.

For these exercises you’ll use the following files:

```
apache/ (the module name)
manifests/
init.pp (contains the apache class)
templates/
vhost/
_file_header.erb (contains the vhost template, managed by Puppet)
```

In this exercise, you’ll modify a template from the Puppet Apache module, specifically `vhost.conf.erb`, to include some simple variables that will be populated by facts about your node.

On the Puppet master, navigate to the modules directory by running 
```
cd /etc/puppetlabs/code/environments/production/modules
```

Now run `ls` to view installed modules: 
Run ls to view  installed modules, and we can see Apache is installed. 
```
[vagrant@puppetmaster modules]$ ls
apache 
```
Open `apache/templates/vhost/_file_header.erb` 
```
sudo vim apache/templates/vhost/_file_header.erb
``` 

It contains the following header
```ruby
# ************************************
 # Vhost template in module puppetlabs-apache
 # Managed by Puppet
 # ************************************
```

Collect the following facts about your agent:
on your `wiki` VM run 
```bash
facter osfamily. 
```
This returns your agent’s OS.

on your `wiki` node also run
```
facter id
``` 

This returns the `id` of the currently logged in user.

Edit the header of `_file_header.erb` so that it contains the following variables for `Facter` lookups:
```ruby
 # ************************************
 # Vhost template in module puppetlabs-apache
 # Managed by Puppet
 #
 # This file is authorized for deployment by <%= scope.lookupvar('::id') %>.
 #
 # This file is authorized for deployment ONLY on <%= scope.lookupvar('::osfamily') %> <%= scope.lookupvar('::operatingsystemmajrelease')     %>.
 #
 # Deployment by any other user or on any other system is strictly prohibited.
 # ************************************
```


Now on `wiki` run the puppet agent 
```bash 
sudo puppet agent --verbose --no-daemonize --onetime
```

At this point, Puppet configures Apache and starts the `httpd` service. When this happens, a default Apache virtual host is created based on the contents of `_file_header.erb`.

On `wiki`, open `/etc/httpd/conf.d/15-default.conf`
```
sudo vim /etc/httpd/conf.d
```

Now you should see something like the following 
```ruby
 # ************************************
 # Vhost template in module puppetlabs-apache
 # Managed by Puppet
 #
 # This file is authorized for deployment by root.
 #
 # This file is authorized for deployment ONLY on CentOS 6.
 #
 # Deployment by any other user or on any other system is strictly prohibited.
 # ************************************
```

As you can see, Puppet has used `Facter` to retrieve some key facts about your node, and then used those facts to populate the header of your `vhost` template.

But now, let’s see what happens when you write your own Puppet code.

## Writing a Puppet module
Puppet modules save time, but at some point you may need to write your own modules.

### Writing a class in a module
In this exercise, you will create a class called `puppet_quickstart_app` that will manage a PHP-based web app running on an `Apache` virtual host.

On the Puppet master, make sure you’re still in the modules directory 
```
cd /etc/puppetlabs/code/environments/production/modules 
```

Then run 
```bash
sudo mkdir -p puppet_quickstart_app/manifests
```
 to create the new module directory and its manifests directory.
Create the `init.pp` file 
```
sudo vim puppet_quickstart_app/manifests/init.pp
```

Add the following: 
```ruby
 class puppet_quickstart_app {

   class { 'apache':
     mpm_module => 'prefork',
   }

   include apache::mod::php

   apache::vhost { 'puppet_quickstart_app':
     port     => '80',
     docroot  => '/var/www/puppet_quickstart_app',
     priority => '10',
   }

   file { '/var/www/puppet_quickstart_app/index.php':
     ensure  => file,
     content => "<?php phpinfo() ?>\n",
     mode    => '0644',
   }

 }
```

You have written a new module containing a new class that includes two other classes: `apache` and `apache::mod::php`.

Note the following about your new class:

As with the `Mediawiki` module we are installing PHP because it is required for the next step. 
`include apache::mod::php` indicates that your new class relies on those classes to function correctly. Puppet understands that your node needs to be classified with these classes and will take care of that work automatically when you classify your node with the `puppet_quickstart_app` class. 
In other words, you don’t need to worry about classifying your nodes with Apache and Apache PHP.

The file `/var/puppet_quickstart_app/index.php` contains whatever is specified by the content attribute. This is the content you will see when you launch your app. Puppet uses the `ensure` attribute to create that file the first time the class is applied.

### Using your custom module in the main manifest
Go to the `manifest` directory
```
cd /etc/puppetlabs/code/environments/production/manifests
```

Your `nodes.pp` file should look like this after you make your changes 
```
  node 'wiki.example.lab' {

   lookup('classes', Array[String], 'unique').include
    class { 'puppet_quickstart_app': }
  }
```


On your `wiki` VM run the following: 
```
sudo puppet agent -t 
```

When the Puppet run is complete, you will see in the agent’s log that a `vhost` for the app has been created and the Apache service `httpd` has been started.

Use a browser to navigate to port 80 of the IP address for `wiki`
`http://172.31.0.202`

If everything is working as expected you should see a PHP page.

Congratulations! You have created a new class from scratch and used it to launch an Apache PHP-based web app. 

## Using Puppet to manage your app
On the `wiki` VM edit  `/var/www/puppet_quickstart_app/index.php`, and change the content to something like, “THIS APP IS MANAGED BY PUPPET!”
```
sudo vim /var/www/puppet_quickstart_app/index.php
```
Refresh your browser, and notice that the PHP info page has been replaced with your new message.
On `wiki` run
```
puppet agent -t --onetime
```

Refresh your browser, and notice that Puppet has reset your web app to display the PHP info page. (You can also see that the contents of `/var/www/puppet_quickstart_app/index.php` has been reset to what was specified in your manifest.)

# Lab Complete