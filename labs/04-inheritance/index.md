# Class inheritance

Let’s say want to create a module that let’s you set up any of the following scenarios on each of your nodes:

* Creates a user (called ‘homer’) only
* Creates a user (called ‘homer’) and textfile.
* Creates a user (called ‘homer’) and ensures a service is running

Note here that the user called “homer” must always exist.

There are 4 main ways to achieve this in your module’s design:

1. create 3 classes, one for each resource. In this scenario, you would have 3 manifests, e.g., `init.pp` (which contains the user_account class), `ensure_file.pp`, and `ensure_service.pp`.
have all resources in a single class in the `init.pp` file, and control whether file or service resource using class parameters.

2. Write a class for each resource, assume the `user` resource is the main class. Also the file and service class also includes a user class.

3. Use inheritance
Option 1 – this is not good, because it relies on the module’s to remember to always call the `user_account` class before the `ensure_file/ensure_service` classes. This means that it makes it possible for user to only ensure `textfile` (by only including the `user_account::ensure_file` class), which is not how the module is supposed to be used, because it isn’t one of the scenarios defined above.

option 2 – Also not good because you have to introduce extra class parameters along with `if/else` logic, which can be avoided.

option 3 – Also not good, because it will result in code duplication, since the user resource has to be defined 3 times, once for each manifest.

Option 4 – this is the best option. But only use inheritance sparingly.

To achieve inheritance, we first have to start with implementing option 1. This means that we end up with:
```bash
[ubuntu@puppetmaster modules]# tree user_account/
user_account/
└── manifests
    ├── ensure_file.pp
    ├── ensure_service.pp
    └── init.pp
```


Let's create a new module `user_accounts`
```bash
sudo mkdir -p /etc/puppetlabs/code/environments/production/modules/user_account/manifests
```

In the new `user_account/manifests` directory we are going to create 3 new files `ensure_file.pp`, `ensure_service.pp` and `init.pp`

In the `init.pp` put the following: 
```ruby
class user_account {

  user { 'homer':
    ensure => 'present',
    uid => '121',
    shell => '/bin/bash',
    home => '/home/homer',
  }

}
```


In `ensure_file.pp` put the following:
```ruby
class user_account::ensure_file {

  file {'/tmp/testfile.txt':
    ensure => file,
    content => "some important data. \n",
  }

}
```

Finally in `ensure_service.pp` let's add: 
```ruby
class user_account {

  service {'sshd':
    ensure => running,
  }

}
```


Note: here we used the “sshd” service just as an example.


Now we need to update our `nodes.pp` to include our new class, right under the `lookup('classes'....)` add 
```ruby
include user_account::ensure_file
```

So that it looks like this: 
```ruby
node 'wiki.example.lab' {

   lookup('classes', Array[String], 'unique').include
   include user_account::ensure_file
}
```

Now let's run the puppet agent on `wiki`
```bash
sudo puppet agent --verbose --no-daemonize --onetime
```


As you can see the user has not been created, but the file has been created.

Now let’s add the inheritance syntax to `ensure_file.pp`, so it looks like this:
```ruby
class user_account::ensure_file inherits user_account {

  file {'/tmp/testfile.txt':
    ensure => file,
    content => "some important data. \n",
  }

}
```

Remove the file that was created 
```bash
sudo rm -rf /tmp/testfile.txt
```

Now run the puppet agent on `wiki` again:
```bash
sudo puppet agent --verbose --no-daemonize --onetime
```

Let's confirm everything was created successfully. 
```bash
sudo grep home /etc/passwd
```

```bash
cat /tmp/testfile.txt
```


The module’s sub class, `ensure_file` inherits the main class. You can think of inheritance, as a way to insert the entire “remote class” to the very beginning of the class in question.

# Lab Complete 