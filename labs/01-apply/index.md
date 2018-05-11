# Puppet apply


In this lab we are going to play around with `puppet apply` and see how it can be used to create resources on-the-fly.  We are then going to use the `puppet resource` command to show the attributes of our previously created resources and add them to a manifest. 

Start out by creating a couple different users by running on the master the  `puppet apply` command. 
```bash
sudo puppet apply -e "user { 'jargyle': ensure => present, }"
sudo puppet apply -e "user { 'tomhanks': ensure => present, }"
sudo puppet apply -e "user { 'bcorgan': ensure => present, }"
```

Now create a group for these users.
```bash
sudo puppet apply -e "group { 'web': ensure => present, }"
```


Now we are going to take the `group` resource and add it to the main manifest file. 

Run the following command
```bash
sudo puppet resource -e group web
```

This should open up a text editor with something similar to the following. 
```ruby
group { 'web':
  			  ensure => 'present',
  			  gid    => '502',
     }
```

Copy this code, save and exit the file. 

Now open the `nodes.pp` file and at the top of the `linux` class paste it, so it looks similar to this.
```ruby
class linux { 
  group { 'web':
    ensure => 'present',
    gid    => 502,
}
```

Save the file and then check the syntax 
```bash
sudo puppet parser validate nodes.pp
```

If it doesn't show any errors than the syntax is good. 

Now let's do the same thing but with the users we created. 

For each user `jargyle`, `tomhanks` and `bcorgan` run the following command replacing `<user>` with one of the users you created.
```bash
sudo puppet resource -e user <user>
```

You should see something similar to this. 

```ruby
user { 'jargyle':
  ensure             => 'present',
  gid                => 501,
  home               => '/home/jargyle',
  password           => '!!',
  password_max_age   => 99999,
  password_min_age   => 0,
  password_warn_days => 7,
  shell              => '/bin/bash',
  uid                => 501,
}
```


We want to add our users to the `web` group so let's add the following.
```ruby
  groups            => 'web',
```

Delete the `gid` line 

Now for each user copy all of the code and save it somewhere, then exit the file.

For each user paste code into the `nodes.pp` file so it looks something like this, with each user defined.
```ruby
user { 'jargyle':
 			  ensure           => 'present',
      gid              => '501',
      home             => '/home/jargyle',
      comment           => 'Judy Argyle',
      groups            => 'web',
      password         => '!!',
      password_max_age => '99999',
      password_min_age => '0',
      shell            => '/bin/bash',
      uid              => '501',
      require         =>  Group['web'],
    }
``` 

Check the syntax
```bash
sudo puppet parser validate nodes.pp
```

If it comes back without any errors log into the `wiki` node and run the agent. 
```bash
sudo puppet agent --verbose --no-daemonize --onetime --noop
```

you may have noticed that nothing actually happened...   this is because we used the `--noop` command.  It does a `dryrun` without actually applying anything. 

If there were no errors you can apply the manifest now. 

```bash
sudo puppet agent --verbose --no-daemonize --onetime --noop
```


Now if that runs to completion you have successfully created 3 new users and added them all to the `web` group!

# Lab Complete 


