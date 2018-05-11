# Puppet Language 

The Puppet DSL is a powerful thing and allows us, not only, to create resources but to set default attributes, iterate through different resources, and define exactly how our environment should look. 

In this lab we are going to play around with some additional language constructs. 

Log into 	`wiki` VM and create a directory for our manifests
```bash
mkdir ~/manifests
```

Now create a manifest named `ntp_temp.pp` for our `ntp.conf` file with the following.
```ruby
#Manage NTP on CentOS and Ubuntu hosts

$ntp_conf = @(END)
#Managed by puppet - do not edit
server us.pool.ntp.org
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org
driftfile /var/lib/ntp/drift
END

case $facts['os']['family'] {
  'RedHat': {
    $ntp_service = 'ntpd'
    $admingroup = 'wheel'
  }
  'Debian': {
    $ntp_service = 'ntp'
    $admingroup = 'sudo'
  }
  default : {
    fail("Your ${facts['os']['family']} is not supported")
  }
}

package { 'ntp':
  before => File['/etc/ntp.conf'],
}

File {
  owner  => 'root',
  group  => $admingroup,
  mode   => '0664',
  ensure => 'file',
}

file { '/etc/ntp.conf':
  content => $ntp_conf,
  notify  => Service['NTP_Service'],
}

service {'NTP_Service':
  ensure    => 'running',
  enable    => true,
  name      => $ntp_service,
  subscribe => File['/etc/ntp.conf'],
}
```

In this file we are using a `Heredoc` to create our `ntp.conf` file.  We could have used a template as well. 

Our `Heredoc` allows us to specify what belongs in the `ntp.conf` by using `@(END)` to populate our file.

You can see we are also using `case` to check the operating system by evaluating the `['os']['family']` fact. 

In this manifest we use `File` to setup default attributes for `ntp.conf`.  

The final thing in the manifest is a `subscribe` attribute that telling the `ntp` service to be restarted if a change is made to `ntp.conf`


Check the syntax

```bash
sudo puppet parser validate ~/manifests/ntp_temp.pp
```

if it comes back without error apply it.
```bash
sudo puppet apply ~/manifests/ntp_temp.pp
```


## Puppet conditionals

Create a manifest utilizing `if` 
```bash
vim ~/manifests/if_condition.pp
```

Add the following:
```ruby
if $facts['os']['family'] == 'RedHat' {
notify { 'Red Hat': }
}
```

Apply it
```bash
sudo puppet apply ~/manifests/if_condition.pp
```
Now let's create a manifest that notifies us if it's NOT a 'Debian' distribution using the `unless` conditional

Create a manifest utilizing `unless`
```bash 
vim ~/manifests/unless_condition.pp
```

Add the following 
```ruby
unless $facts['os']['family'] == 'Debian' {
notify { 'RedHat': }
}
```

Apply it
```bash
sudo puppet apply ~/manifests/unless_condition.pp
```

Now following the procedure from above create  two manifests 

one using `else` and the other  `elsif` 


## Lambda iteration
Now let's create a few different files by iterating over a `lambda` array.

Create `~/manifests/iteration.pp` with the following
```bash
$files = ['test1', 'test2', 'test3', 'test4', 'test5']

$files.each |String $files| {
  file {"/tmp/${files}":
    ensure => present,
  }
}
```

As you can see we've created an array `files` with 5 items in it.  We then iterate over all the items in the array and use the `file` resource to ensure they are created. 


Check the syntax

```bash
sudo puppet parser validate ~/manifests/iteration.pp
```

if it comes back without error apply it.
```bash
sudo puppet apply ~/manifests/iteration.pp
```

Now confirm the files were created 
```bash
ls -l /var/tmp/test*
```
 

# Lab Complete 