---
layout: post
title: "Chef Introduction [Part 1]"
description: ""
category: 
tags: []
---
{% include JB/setup %}
##Chef

###When setup or upgrade servers, Chef solve problems: 

1. Very hard to keep track of what’s been done.
2. Slow.
3. Not scale.

###Terminology

#####Recipe:

Chef definition for installing a single component, e.g. Ruby, mysql-server, Monit etc

#####Cookbook:

A selection of recipes, so for example a “mysql” cookbook might include a recipe for a MySQL server and another one for a MySQL client.

#####Node:
A remote server we’re provisioning

#####Role:
A combination of recipes which when applied to a node, allows it to perform a particular role.
For example a “Postgres Server” role might include recipes for installing postgres-server as well as installing and configuring the firewall and setting up suitable monitoring for the server process.
You can sort of think of a role as an equivalent to a mixin in ruby. It allows you to create a piece of re-usable functionality that can later be applied to many nodes.

#####Attribute:
An attribute is a parameter, usually in the form of a key value pair in either a node or role definition which allows us to customise the behaviour of a recipe. Typical things attributes might be used to control include configuration variables and the versions of packages which are installed.

#####Data Bag:
A JSON file which contains metadata used by recipes, for example lists of users to be created and the valid public keys for authenticating as them.

#####Chef Repository:
A collection of node and role definitions.


###Set up a Project
1. Create dir

		mkdir chef_projects
		cd chef_projects
    
    
2. Create Gemfile	

		source 'https://rubygems.org'
		gem 'knife-solo', '0.4.2'
		gem 'chef', '~> 11.16.0'
		gem 'chef-zero', '2.2'
		gem 'berkshelf', '~> 3.1.5'

3. Install

		bundle install
	

###Create a Chef repository	

1. Create repository
		
		bundle exec knife solo init rdr_redis_example
	
	
2. Copy Gemfile & Gemfile.lock to the newly created repository

		cp Gemfile rdr_redis_example
		cp Gemfile.lock rdr_redis_example
	
####Directory Structure

	├── .chef
	│  └── knife.rb
	├── .gitignore
	├── Berksfile
	├── Gemfile
	├── Gemfile.lock
	├── cookbooks
	├── data_bags
	├── environments
	├── nodes
	├── roles
	└── site-cookbooks

#### knife.rb

Knife is the command line utility which allows us to interact with Chef on our remote server.

Documentation for the full range of options available for use in knife.rb is available at:

http://docs.opscode.com/config_rb_knife.html

#### cookbooks vs site-cookbokks

The config: ```knife[:berkshelf_path] = "cookbooks"``` means that the cookbooks directory will be used to store copies of cookbooks managed by Berkshelf (it’s like Bundler for Chef cookbooks).

Not to use the cookbooks directory for anything not managed by Berkshelf because
it is wiped wiped automatically when we start provisioning.

**If we wish to create cookbooks which are not managed by Berkshelf, we will always do this in the site-cookbooks directory.**

### Setting up a node

1. Copy public key to remote server.
  	  		
		ssh-copy-id root@YOURSERVERIP
  
  
2. Create a skeleton node definition
    		
		bundle exec knife solo prepare root@YOURSERVERIP  
  
  
  It do two things:
  1. the deb package for chef is being downloaded and installed on the remote machine
  2. a node config is generated locally in the folder nodes with the filename YOURSERVERIP.json.


##Sample: Install Redis server
  
###Generate Cookbook


	knife cookbook create redis-server -o site-cookbooks


####Structure of a Cookbook

  
	├── attributes
	├── definitions
	├── files
	│   └── default
	├── libraries
	├── providers
	├── recipes
	│   └── default.rb
	├── resources
	├── templates
	│   └── default
	├── CHANGELOG.md
	├── README.md
	└── metadata.rb
  
####metadata.rb
This file contains details about the purpose of the cookbook as well as any dependencies and the version.
####The recipes directory
This directory contains individual recipes. All cookbooks should have a default recipe defined in recipes/default.rb but may also declare others.
####The templates directory
When we want to create a file on the remote server, we’ll create erb files **within a subfolder(represents different OS)** of this directory which chef will then convert into a remote file. The advantage of using erb here is that it means we can dynamically generate and interpolate values based on attributes.
####The attributes directory
Attributes are values most commonly set in a node or role definition which allow us to customise the behavior of a recipe. Common uses would include choosing the version of a package to be installed and customising configuration file values.
In order to set default values for these attributes, we can create a file default.rb within the attributes folder of a cookbook. If we do not specify a value for an attribute in a node or role definition, the values from this file will be used.

###1. Add dependency on 'apt' cookbook
'apt' cookbook Github: [https://github.com/opscode-cookbooks/apt.git](https://github.com/opscode-cookbooks/apt.git "https://github.com/opscode-cookbooks/apt.git")

#### Berkshelf
Berkshelf provides exactly functions as Bundler in rails:
1. Manage dependencies 
2. Generate Berksfile.lock

#####rdr_redis_example/Berksfile
---

	source "https://api.berkshelf.com"

	cookbook 'apt', github: 'opscode-cookbooks/apt'

Execute command:

	bundle exec berks install

Add dependency in newly created cookbook: `redis-server`'s metadata

#####rdr_redis_example/site-cookbooks/redis-server/metadata.rb
---

	name             'redis-server'
	maintainer       'YOUR_COMPANY_NAME'
	maintainer_email 'YOUR_EMAIL'
	license          'All rights reserved'
	description      'Installs/Configures redis-server'
	long_description IO.read(File.join(File.dirname(__FILE__), 'README.md'))
	version          '0.1.0'
	depends          'apt' #Add dependency on 'apt'

####Use DSL provided by `apt` cookbook to define 'Lightweight Resource Providers(LWRP)'

#####rdr_redis_example/site-cookbooks/redis-server/recipes/default.rb 
---

	apt_repository 'redis-server' do
		uri          'ppa:chris-lea/redis-server'
		distribution node['lsb']['codename']
	end

####Using Chef DSL to define package to install
Documentation: [https://docs.getchef.com/resource_package.html](https://docs.getchef.com/resource_package.html "https://docs.getchef.com/resource_package.html")

#####rdr_redis_example/site-cookbooks/redis-server/recipes/default.rb 
---

	apt_repository 'redis-server' do
		uri          'ppa:chris-lea/redis-server'
		distribution node['lsb']['codename']
	end

	package 'redis-server'

###2. Add flexibilty with attributes
#####rdr_redis_example/site-cookbooks/redis-server/attributes/default.rb 
---

	default['redis-server']['appendonly'] = 'no'
	default['redis-server']['additional_configuration_values'] = {}
	default['redis-server']['bind'] = '127.0.0.1'
	default['redis-server']['daemonize'] = 'yes'
	default['redis-server']['databases'] = 16
	default['redis-server']['dbfilename'] = 'dump.rdb'
	default['redis-server']['dir'] = '/etc/redis/'
	default['redis-server']['logfile'] = '/var/log/redis/redis-server.log'
	default['redis-server']['loglevel'] = 'notice'
	default['redis-server']['package'] = 'redis-server'
	default['redis-server']['pidfile'] = '/var/run/redis/redis-server.pid'
	default['redis-server']['port'] = '6379'
	default['redis-server']['ppa'] = 'ppa:chris-lea/redis-server'
	default['redis-server']['rdbpcompression'] = 'yes'
	default['redis-server']['save'] = [
		'900 1',
		'300 10',
		'60 10000'
	]
	default['redis-server']['timeout'] = 300

####Access attributes from recipes
Attributes can be accessed from `node` variable.

#####rdr_redis_example/site-cookbooks/redis-server/recipes/default.rb 
---

	apt_repository 'redis-server' do
		uri          node['redis-server']['ppa']
		distribution node['lsb']['codename']
	end

	package node['redis-server']['package']


####Access attributes from templates
Attributes can be accessed from `node` variable.

#####rdr_redis_example/site-cookbooks/redis-server/templates/default/redis_conf.rb
---

	daemonize <%= node['redis-server']['daemonize'] %>

	pidfile <%= node['redis-server']['pidfile'] %>

	logfile <%= node['redis-server']['logfile'] %>

	port <%= node['redis-server']['port'] %>

	bind <%= node['redis-server']['bind'] %>

	timeout <%= node['redis-server']['timeout'] %>

	loglevel <%= node['redis-server']['loglevel'] %>

	databases <%= node['redis-server']['databases'] %>

	<% node['redis-server']['save'].each do |save_entry| %>
		<%= "save #{save_entry}" %>
	<% end %>

	rdbcompression <%= node['redis-server']['rdbcompression'] %>

	dbfilename <%= node['redis-server']['dbfilename'] %>

	dir <%= node['redis-server']['dir'] %>

	appendonly <%= node['redis-server']['appendonly'] %>

	<% node['redis-server']['additional_configuration_values'].each do |key, value| %>
 		<%= "#{key} #{value}" %>
	<% end %>


###3. Using templates in recipe

#####rdr_redis_example/site-cookbooks/redis-server/recipes/default.rb 
---

	.....
	.....

	template "/etc/redis/redis.conf" do
	  owner "redis"
	  group "redis"
	  mode "0644"
	  source "redis.conf.erb" # use redis.conf.erb in templates/default directory.
	end

	directory "/etc/redis" do
	  owner 'redis'
	  group 'redis'
	  action :create
	end

The internals of how Chef updates file is smart. It will compare the desired state of the file with the actual state of the file on the remote system, and only update the file on the remote system if the two are different.

###4. Restart redis-server only when config changes
Add `execute` DSL block

#####rdr_redis_example/site-cookbooks/redis-server/recipes/default.rb 
---

	.....
	.....
	execute "restart-redis" do
	  command "/etc/init.d/redis-server restart"
	  action :nothing #This command won't run when execution of the recipe reachs here.
	end


Notify the `execute` block when the config changes.

#####rdr_redis_example/site-cookbooks/redis-server/recipes/default.rb 
---
	.....
	.....
	template "/etc/init.d/redis-server" do owner "redis"
	  group "redis"
	  mode "0755"
	  source "redis-server.erb"
	  notifies :run, "execute[restart-redis]", :immediately # Notify restart-redis execution block
	end

###5. Updating the node

	bundle exec knife solo bootstrap root@YOURSERVERIP

##Reference:
Complete Code: 
[https://github.com/TalkingQuickly/redis-server/tree/5-more_flexible_attributes](https://github.com/TalkingQuickly/redis-server/tree/5-more_flexible_attributes "https://github.com/TalkingQuickly/redis-server/tree/5-more_flexible_attributes")

