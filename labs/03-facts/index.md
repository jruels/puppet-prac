# Facts
In this lab we are going to play around with `facter` and explore the different types of facts Puppet has.  

There are `core` facts which are built into Puppet and provide a ton of useful information.  Just a few of the commonly used `core` facts are: 
* ['os']
* ['networking']
* ['memory']
Each of these can be used in manifests out of the box. 

Let's use the `facter` command to see what facts are available, and discuss how we can use them. 


Start off by listing all of the `core` facts on the `wiki` server
```bash
sudo facter -p 
```

you should see quite a bit of output.

Now let's find out the IP address
```bash
sudo facter -p networking.ip 
```

Alright, what if we need to know how much memory is available.
```bash
sudo facter -p memory
```

and to drill down into the total amount on the system. 
```bash
sudo facter -p memory.system.total
```

As we've seen previously it's also possible to use `facts` in our manifests. 

We can do a check to determine which Operating System our VM is prior to running the manifest and based on the fact define which package to install or where the config file should be located. An example of which is below. 

```ruby
class mediawiki {
  $phpmysql = $osfamily ? {
    'redhat'  => 'php-mysql',
    'debian'  => 'php5-mysql',
    default   => 'php-mysql',
  }

  package { $phpmysql:
    ensure  => 'present',
  }

}
```


## External facts 
What happens if we want to find out information about something that the `core` facts don't provide?  We can write our own facts! 

External facts can be written in any language and can even be a simple `yaml` or `json` file with static data in them. 

Let's start off by creating an external fact written in `bash` which will do a simple `hostname` command and use `cut` to tell us this is an `example` server. 

On your master create your fact
```bash
sudo vim /opt/puppetlabs/facter/facts.d/my_environment.sh
```

Inside of this file put the following 
```bash
#!/usr/bin/env bash
[ "$(hostname -f  | cut -d. -f2)" ]
echo "server_environment=example"
```

Now that we've created our external fact  we need to give it execute permissions.
```bash
chmod +x /opt/puppetlabs/facter/facts.d/my_environment.sh
```


Confirm it is working with `facter`
```bash
sudo facter -p server_environment
```

You should see value of `example` returned, which means that our brand new external fact is working as expected! 

Now let's add this to a simple manifest and run the agent to see if it works as expected. 
```bash
mkdir ~/manifests
sudo vim ~/manifests/environment_check.pp
```

Add the following to `environment_check.pp`
```ruby
if $facts['server_environment'] == 'example' {
    package { 'tree':
      ensure => 'present',
    }
  }
```

Check the syntax to make sure everything looks good 
```bash
sudo puppet parser validate ~/manifests/environment_check.pp
```
 
If it doesn't output any messages then your syntax is correct. 

Now run the manifest
```bash
sudo puppet apply ~/manifests/environment_check.pp
```

If everything was successful you should have seen output with the following.
```
Notice: /Stage[main]/Main/Package[tree]/ensure: created
```

To confirm it was installed you can run 
```bash
tree ~/manifests
```

Output should show you something like:
```bash
/home/vagrant/manifests
└── environment_check.pp

0 directories, 1 file
```

## Custom facts 
Now that we've used `core` facts and `external` facts let's create a `custom` fact. 

Because we are going to define the location of our custom facts through an environment variable we need to become the root user first.
```bash
sudo su - 
```

After becoming the root user we need to create a directory for our custom fact 
```bash
mkdir /opt/puppetlabs/my_facts
```


Set the  `FACTERLIB` variable to the newly created directory. 
```bash
export FACTERLIB="/opt/puppetlabs/my_facts"
```
 

Our custom fact is going to run a shell command and then return the architecture of our server. 

The command it will be running is `uname --hardware-platform`

We are going to give our fact a new of `hardware_platform` .
Now let's create our custom fact
```bash
vim /opt/puppetlabs/my_facts/hardware_platform.rb`
```

In this file add the following:
```ruby
Facter.add('hardware_platform') do
  setcode do
    Facter::Core::Execution.execute('/bin/uname --hardware-platform')
  end
end
```

The `Facter.add` function provides the name of our fact `hardware_platform` and then we use `.execute` to run our shell command.

Now that we've created our custom fact let's test it with `facter` and make sure it returns the result we're looking for. 
```bash
facter -p hardware_platform
```

You should get a return 
```
x86_64
```

# Lab Complete 