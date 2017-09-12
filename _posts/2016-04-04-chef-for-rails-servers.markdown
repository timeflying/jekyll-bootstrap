---
layout: post
theme: twitter
title: Chef for Rails servers
---

## Basic Server Setup

### Useful Tools

`look-and-feel_tlq` cookbook
https://github.com/TalkingQuickly/look_and_feel-tlq
Includes: htop, zip, vim...

### Unattended upgrades
Automatic upgrades can ensure system remains upda to date and secure, but may break existing functionality,
It's good practice to automatic upgrads for security related package updates only.

#### Configuring

##### roles/server.json 

	...
	"default_attributes": {
	  "apt" : { 
    	"unattended_upgrades" : {
	      "enable" : true,
    	  "allowed_origins" : [
        	    "${distro_id} stable",
            	"${distro_id} ${distro_codename}-security"
	    ],
    	  "automatic_reboot" : false,
      	"mail": "..."
      	}
	},
	...

### Automatic System Time via NTP

The server role includes `recipe[ntp::default]`, in its run list. This has the effect of installing NTP which will periodically synchronize the system clock with a remote time server.

### Locales

#### Why Locales Matter
1. If we are using Postgres as our database provider, it will use the systems locale when it creates the initial database cluster. If we are using different locales in development and production, our systems are not identical and will behave differently in some respects.
2. SSH client will send LC_* variable to server. This may be problematic:
• A server is setup with only the locale en_US.utf8
• Our locale developmen machine is setup with the locale en_GB.utf8

#### Configuring

Two cookbooks which concern locales:

##### roles/server.json 


	"locales" : {
	  "packages" : ["locales"],
	  "default" : "en_US.utf8" 
	},
	

This defines the package(s) which provide locale functionality, for Ubuntu this is always locales. It then allows us to choose the locale which should be installed and set as the default locale.

##### roles/server.json 


	"look_and_feel-tlq" : {
	  // any extra locales we want available. Useful if your 
	  // local dev machine uses a locale which doesn't match 
	  // the servers locale.
	  "additional_locales" : ["en_GB.utf8"]
	},

This uses the LWRP defined by the above locales cookbook to allow for the installation of multiple other locales alongside the default one.

## Security

### Common pitfalls
Mistake 1 - Not Updating Gems
Mistake 2 - Hard coding credentials
Mistake 3 Re-using Passwords

### SSH Hardening

##### roles/server.json 


	"openssh" : {
    	  "server" : {
        	"password_authentication" : "no",
	        "challenge_response_authentication" : "no",
    	    "permit_empty_passwords" : "no",
        	"use_pam" : "no",
	        "x11_forwarding" : "no",
    	    "permit_root_login" : "yes"
	      }
    }


### Firewall

`ufw` cookbook which stands for “Uncomplicated Firewall” :  https://github.com/opscode-cookbooks/ufw

#### Configuring

##### roles/ngnix.json 

    "default_attributes": {
      "firewall" : {
        "rules" : [
          {"allow http on port 80" : {"port" : 80}}
        ]
      }
    }

### Users

`users` cookbook: https://github.com/chef-cookbooks/users

##### data_bags/users/deploy.json


	{
	  "id": "deploy",
	  // generate this with: openssl passwd -1 "plaintextpassword" 
	  "password":     "$1$jil6kjrT$iZ5o0DkBc8rBOljXxo.4j1",
	  // the below should contain a list of ssh public keys which should // be able to login as deploy
	  "ssh_keys": [
    	"rsa_public_key"
	  ],
	  "groups": [ "sysadmin"], 
	  "shell": "\/bin\/bash"
	}

The `sysadmins` recipe which is included in the server role will search for any user in the sysadmin group and add them to the system.

### Sudo

`sudo` cookbook : https://github.com/ opscode-cookbooks/sudo 
Included in the server role `recipe[sudo::default]`.

##### roles/server.json 


    "authorization": {
      "sudo": {
        // everyone in the group sysadmin gets sudo rights
        "groups": ["sysadmin"],
        // the deploy user specifically gets sudo rights
        "users": ["deploy"],
        // whether a user with sudo rights can execute sudo
        // commands without entering their password.
        "passwordless": true
      }
    },


## Ruby & Gem dependencies

### How rbenv works
Rbenv will then determine which ruby version to execute the command with from the following sources, in order:

1. The RBENV_VERSION environment variable
2. The first .ruby-version file found by searching the directory which the script being
executed is in, and all parent directories until reaching the filesystem root
3. Thefirst.ruby-versionfilefoundbysearchingthecurrentworkingdirectoryandallparent
directories until reaching the filesystem root
4. ∼/.rbenv/version which is the users global ruby. Or /usr/local/rbenv/version if it is a
system wide install.
5. If none of these are available, rbenv will default to the version of Ruby which would have been run if rbenv were not installed (e.g. system ruby)

### Configuring

`chef-rbenv` cookbook

##### roles/rails-app.json 


	"default_attributes": {
    	"rbenv":{
	      "rubies": [
    	    "2.1.2"
	      ],
    	  "global" : "2.1.2",
	      "gems": {
    	    "2.1.2" : [
        	  {"name":"bundler"}
	        ]
	      }
	    }
	  },

The rbenv::system performs a system wide install of rbenv, this means that it’s installed to /usr/local/rbenv rather than ∼/rbenv. This is generally my preference as it reduces issues caused by, for example, cron jobs running as root.

### Gem Dependencies

The rails-server role also includes `recipe[rails_gem_dependencies-tlq::default]` which is a very simple cookbook I created to install packages required when compiling native extensions for common gems.

##### rails_gem_dependencies-tlq/recipes/default.rb


	package 'curl'
	package 'libcurl3'
	package 'libcurl3-dev'
	package 'libmagickwand-dev'
	package 'imagemagick'
	apt_repository("node.js") do
	  uri "http://ppa.launchpad.net/chris-lea/node.js/ubuntu"
	  distribution node['lsb']['codename']
	  components ["main"]
	  keyserver "keyserver.ubuntu.com"
	  key "C7917B12"
	end
	package 'nodejs'

it installs some standard packages then adds a ppa which provides an up to date version of nodejs which we can use as our javascript runtime when compiling assets on the remote server.

## Monit

Monit can monitor system parameters such as processes and network interfaces. If a process fails or moves outside of a range of defined parameters, Monit can restart it, if a restart fails or there are too many restarts in a given period, Monit can alert us by email. If needed, using email to SMS services, we can have these alerts delivered by text message.

 1. `monit-tlq` cookbook
 2. `monit_configs-tlq` cookbook
 
### Monit-tlq

This cookbook takes care of installing and configuring Monit and contains only one recipe default.

It begins by installing the Monit package and updates Monits main configuration file `/etc/- monit/monitrc` from the template `monit-rc.erb`.

##### monit-tlq/templates/default/monit-rc.erb


	set daemon <%= node[:monit][:poll_period] || 30 %>

This sets the interval in seconds which Monit will performs its checks at. In Monit terminology this is known as a cycle.

##### monit-tlq/templates/default/monit-rc.erb


	set logfile syslog facility log_daemon

This line tells Monit to log error and status messages to /var/log/syslog.

##### monit-tlq/templates/default/monit-rc.erb


	<% if node[:monit][:enable_emails] %>
	  <% if node[:monit][:notify_emails] %>
	    <% node[:monit][:notify_emails].each do |email| %>
	      <% if node[:monit][:minimise_alerts] %>
	set alert <%= email %> but not on {instance, pid, ppid, resource}
	      <% else %>
	set alert <%= email %>
	      <% end %>
	    <% end %>
	  <% end %>

	<% if node[:monit][:mailserver] %>
	set mailserver "<%= node[:monit][:mailserver][:host] %>' port <%= node[:monit][:mailserver][:port] %>"
		  username "<%= node[:monit][:mailserver][:username] %>"
		  password "<%= node[:monit][:mailserver][:password] %>"
		  using tlsv1
		  with timeout 30 seconds
		  using hostname "<%= node[:monit][:mailserver][:hostname] %>"
		<% end %>
	<% end %>

This section sets up a mailserver and the users which should be alerted when appropriate conditions are met.
The addition of `but not in {instance, pid}` prevents emails from being sent due to events which usually don’t require any manual intervention.

##### monit-tlq/templates/default/monit-rc.erb


	set httpd port 2812 and
	    use address localhost
	    allow localhost
	    <% if node[:monit] && node[:monit][:web_interface] && node[:monit][:web_interface][:allow] %>
           allow "<%= node[:monit][:web_interface][:allow][0] %>":"<%=node[:monit][:web_interface][:allow][1]%>"

    <% end %>

The username and password (basic auth) are set in the final allow line.

##### monit-tlq/templates/default/monit-rc.erb


	include /etc/monit/conf.d/*.conf

Finally we tell Monit to include any other configuration files in `/etc/monit/confi.d/*.conf`


The processes and services we’re going to monitor fall into two categories; system components and app components.
System components are those which are installed when provisioning, for example Nginx, our database, Redis, memcached and ssh.
App components are generally processes specific to the Rails app(s). like : unicorn, Sidekiq.

If chef is used to install something (generally system components) then a suitable Monit configuration should be added by chef.

If a component is added via Capistrano (Rails apps and background workers) then the Monit configuration should be defined within the app and managed from Capistrano

### System level monitoring

	check system localhost
	  if loadavg (1min) > 4 then alert
	  if loadavg (5min) > 3 then alert
	  if memory usage > 75% then alert
	  if cpu usage (user) > 70% for 5 cycles then alert
	  if cpu usage (system) > 30% for 5 cycles then alert
	  if cpu usage (wait) > 20% for 5 cycles then alert

The first three are perform a check, if the criteria are met then “alert”

This second three tells Monit will only perform the specified action (in this case alert) if the conditions are met for 5 checks in a row.

#### Load Average

The load average refers to the systems load, taken over three time periods, 1, 5 and 15 minutes.
1.0 means the system is loaded to exactly its maximum capacity for 1 core.
On systems with more than one core, we can roughly multiply your maximum acceptable load by the number of cores.

#### Monitoring Pids

A typical Monit definition for monitoring a pidfile

	check process nginx with pidfile /var/run/nginx.pid
	  start program = "/etc/init.d/nginx start"
	  stop program = "/etc/init.d/nginx stop"
	  if 15 restarts within 15 cycles then timeout

We define a start command and a stop command. On each check, if process is not found to be running, Monit will execute the start command.
The last line means that if there have been 15 attempts to restart a process in the last 15 cycles, then stop trying to restart it.

#### Finding Pidfiles

If there is no mention of the pidfile in the application, the next place to look is /run which is the “standard” location for pidfiles in Ubuntu

#### Pidfile Permissions

	# make sure the pid destination is writable
	mkdir -p /var/run/an_application/
	chown application_user:application_user /run/an_application

#### Monitoring Ports

	if failed host 127.0.0.1 port 80 then restart

Monit will attempt to establish a connection 12.0.0.1:80. If a connection cannot be established, then it will attempt to restart the process.

#### Free Space Monitoring

	check filesystem rootfs with path / 
  		if space usage > 80% then alert

#### Avoiding alert spamming

For example:

	set alert <%= email %> but not on {instance, pid}

This means that globally alerts will be sent for all events except for instance and pid changes.

#### Serving the web interface of Monit  with Nginx

Since allow public IPs to access Monit Web Interface is dangerous, we use nginx as proxy.
Ngnix config:

	server {
	  listen 80;
	  server_name machine-name.monit.example.com; location / {
	    proxy_pass http://127.0.0.1:2812;
	    proxy_set_header Host $host;
	    }
	  }
	server {
	  listen 80;
	  server_name machine2-name.monit.example.com; location / {
	    proxy_pass http://168.1.1.5:2812;
	    proxy_set_header Host $host;
	  }
	}

The above definition show that Nginx is serving both the Monit interface for itself (on 127.0.0.1) and for the second server on 168.1.1.5.

We allow Monit only allow internal IP to access & configure Ngnix to transfer the requests according their requesting hostname.

## Ngnix

Nginx proxies requests back to our application server (Unicorn). 

##### roles/nginx-server.json


	{
		"name": "nginx-server",
		"description": "Nginx server",
		"default_attributes": {
			"firewall" : {
	        "rules" : [
	          //allow access to port 80 from any server
    	      {"allow http on port 80" : {"port" : 80}}
	        ]
	      	},
	      	"nginx" : {
    	    	//Disable the default virtualhost
	        	"default_site_enabled" : false
    	  	}
    	},
    	"json_class": "Chef::Role",
    	"run_list": [
    		//adds the ppa for the current stable version of nginx
    		"recipe[nginx::repo]",
    		// install nginx using the native package provided by the OS’s package manager
    		"recipe[nginx::default]",
    		//Monit configuration for Nginx
    		"recipe[monit_configs-tlq::nginx]",
    		//UFW cookbook
    		"recipe[ufw::default]"
    		],
    	"chef_type": "role"
    }


#### Virtual Hosts (Should handled by applications deployment process)

Anything specific to a single application being deployed, should be handled by the applications deployment process rather than the server provisioning process. 
This allows a single server to to be used to host multiple applications without provisioning changes which may be disruptive to all applications on it.

#### Generated Monit configuration for Ngnix(the `recipe[monit_configs-tlq::nginx]` cookbook )

	check process nginx with pidfile /var/run/nginx.pid
	  start program = "/etc/init.d/nginx start"
	  stop program = "/etc/init.d/nginx stop"
	  if failed host 127.0.0.1 port 80 then restart
	  if 15 restarts within 15 cycles then timeout


## MySQL

For a simple installation using the ‘mysql-server’ role, the only parameters it is necessary to set in the node definition are:

##### nodes/YOURSERVERIP.json


	"mysql": { 
	  "server_root_password":"your_password",
	  "server_debian_password":"your_password",
	  "server_repl_password":"your_password"
	}

It’s important to note that the chef recipe can only set the passwords when it first installs MySQL. This cannot be used to change the passwords post installation. This also means that if, for any reason, the mysql-server package has already been installed when the mysql::server recipe is run, it is likely to fail. If therefore you run into strange permissions errors when chef reaches MySQL, check carefully that no other recipe is installing the mysql-server package as a dependency.

#### Generated Monit configuration for MySQL

	check process mysql with pidfile /var/run/mysqld/mysqld.pid
	  group database
	  start program = "/etc/init.d/mysql start"
	  stop program = "/etc/init.d/mysql stop"
	  if failed host 127.0.0.1 port 3306 then restart
	  if 15 restarts within 15 cycles then timeout


## Redis & Memecached

##### roles/redis-server.json


	{
		"name": "redis-server",
		"description": "Redis server",
		"default_attributes": {
			"redis-server": {
			}
		},
		"json_class": "Chef::Role",
		"run_list": [
			"recipe[redis-server::default]",
			"monit_configs-tlq::redis-server"
			],
		"chef_type": "role"
	}

If we want to limit its maximum resource usage. We can add the following to either the redis-server role or the node definition:


	'redis-server': {
		'additional_configuration_values': {
		'maxmemory': 419430400,
		'maxmemory-policy': 'allkeys-lru',
		'maxmemory-samples': '10'
		}
	}

It means 10 of keys will be selected randomly and the least recently used of these will be discarded.
 
### Monitoring

##### monit_configs-tlq/templates/default/redis-server.conf.erb


	check process redis with pidfile /var/run/redis/redis-server.pid
		group database
		start program = "/etc/init.d/redis-server start"
		stop program = "/etc/init.d/redis-server stop"
		if failed host 127.0.0.1 port 6379 then restart
		if 15 restarts within 15 cycles then timeout


Reference:
Monit doc: http://mmonit.com/monit/documentation/monit.html#alert_messages

