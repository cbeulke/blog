---
layout: 	post
title: 		"Node.js und MySQL mit Vagrant"
date:   	2016-11-18 12:00:00 +0100
comments: 	true
---
Setting up a development environment with Vagrant is a good solution to keep your system clean. For developing my Node.js applications it has many advantages. So i set up an own virtual machine for each of my projects to have a consistent environment for them while also being able to use my central tools.

## Preconditions

- [Vagrant][vagrant-site]

## Init a new Vagrant project

Create a new project directory and init the `Vagrantfile`.

{% highlight bash %}
mkdir project
cd project
vagrant init
{% endhighlight %}

Customize your `Vagrantfile`. There are many predefined but outcommented options wich i don't need, so i delete them to have a clean configuration file.

{% highlight ruby %}
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
  config.vm.provision :shell, path: "bootstrap.sh"
  config.vm.network :forwarded_port, guest: 3000, host: 3000
end
{% endhighlight %}

A vagrant virtual machine needs to define its own operating system so i choose [Ubuntu][ubuntu-site]. Provisioning is done through a bootstrap script which will be described later. The last configuration ensures port 3000 to be open for communication with the outer world (my native system).

## Assembling the bootstrap.sh

The following components shall be installed in my virtual machine.

- Node.js
- NPM (node package manager)
- Gulp.js
- a MySQL server instance

I create a `bootstrap.sh` file which will be used for provisioning my virtual machine.

{% highlight bash %}
touch bootstrap.sh
{% endhighlight %}

After opening the newly created bootstrap.sh i make it runnable and update the Ubuntu package manager. 

{% highlight bash %}
#!/usr/bin/env bash
apt-get update
{% endhighlight %}

### Installing Node.js, NPM and Gulp.js

The next step is adding commands for installing Node.js. This step requires `curl` which i install first.

{% highlight bash %}
apt-get install -y curl
curl -sL https://deb.nodesource.com/setup_6.x | bash -
apt-get install -y nodejs
{% endhighlight %}

There is no further need to install NPM, it is contained alongside with Node.js.

For using Gulp.js as my preferred build tool i ínstall it via npm globally.

{% highlight bash %}
npm install -g gulp-cli
{% endhighlight %}

### MySQL server installation

The next step is installing the MySQL database. It is a bit tricky to install MySQL headless so i had to make [some configuration][mysql-installation] in my system first. Otherwise the root password has to be entered during the installation process which shall be done automatically.

{% highlight bash %}
debconf-set-selections <<< 'mysql-server-5.5 mysql-server/root_password password rootpass'

debconf-set-selections <<< 'mysql-server-5.5 mysql-server/root_password_again password rootpass'
{% endhighlight %}

That ensures the system to install MySQL with the root password `rootpass`.

{% highlight bash %}
apt-get install -y mysql-server-5.5
{% endhighlight %}

After a while the installation should be done successfully.

### Initializing the database

The last thing left to do is creating an exclusive database user for our application development and also creating a database. As an extra feature i will check the `data/` directory of our project folder each time Vagrant provisions a new virtual machine so i will be able to provide an init sql script for some predefined database structure.

{% highlight bash %}
if [ ! -f /var/log/databasesetup ]; then
	echo "CREATE USER 'nodeuser'@'localhost' IDENTIFIED BY 'nodepass'" | mysql -uroot -prootpass
	echo "CREATE DATABASE node" | mysql -uroot -prootpass
	echo "GRANT ALL ON node.* TO 'nodeuser'@'localhost'" | mysql -uroot -prootpass
	echo "flush privileges" | mysql -uroot -prootpass

	touch /var/log/databasesetup

	if [ -f /vagrant/data/init.sql ]; then
		mysql -uroot -prootpass node < /vagrant/data/init.sql
	fi
fi
{% endhighlight %}

## Test run

I can start my virtual machine via `vagrant up`. After successfully provisioning i am able to connect to it with `vagrant ssh`. Now i have full access to the freshly installed ubuntu and have access to mysql as well as to Node.js, NPM and Gulp.js.

The directory `/vagrant` contains all files in the project folder so this is the connection to my native system. Now i am able to edit my project files with the tools installed in my native system while having a strictly defined runtime environment for my project.

To clean the virtual box run `vagrant destroy` and then `vagrant up` again. The virtual machine will get installed freshly.

Here you can find the complete [`bootstrap.sh`][bootstrap-gist].

[vagrant-site]: https://www.vagrantup.com/downloads.html
[ubuntu-site]: https://atlas.hashicorp.com/ubuntu/boxes/xenial64
[mysql-installation]: http://www.thisprogrammingthing.com/2013/getting-started-with-vagrant/
[bootstrap-gist]: https://gist.github.com/cbeulke/3a3611d6e06a457e081332e6ff0c4a56