---
layout: post
title: "Chef Introduction [Part 2]"
description: ""
category: 
tags: []
---


{% include JB/setup %}

##Node & Role Definitions

Recap:


1. #### Directory Structure

```   	
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
```
3. YOURSERVERIP.json has been created in nodes directory when setting up a node.

### Node Definitaion

Override default attributes by node specific definition:

##### rdr_redis_example/nodes/YOURSERVERIP.json
---
```json
	{
      "run_list": [
        "recipe[redis-server::default]"
      ],
      "automatic": {
        "ipaddress": "YOURSERVERIP"
      }
      ,"redis-server": {
        "port": "6380",
        "additional_configuration_values": {
        "maxmemory": "419430400",
        "maxmemory-policy": "allkeys-lru",
        "maxmemory-samples": "10"
        }
      }
    }
```
Apply changes:

    knife solo cook root@YOURSERVERIP

### Role Definition

##### rdr_redis_example/roles/redis-cache-server.json
```json
{
  "name": "redis-cache-server",
  "description": "Redis cache server, automatically discard keys",
  "json_class": "Chef::Role",
  "default_attributes": {
    "redis-server": {
      "port": "6380",
      "additional_configuration_values": {
        "maxmemory": "419430400",
        "maxmemory-policy": "allkeys-lru",
        "maxmemory-samples": "10"
      }
    }
  },
  "chef_type": "role",
  "run_list": [
    "recipe[redis-server]"
  ]
}
```
Add role to node definition:

##### rdr_redis_example/nodes/YOURSERVERIP.json
```json
{
  "run_list": [
    "role[redis-cache-server]"
  ],
  "automatic": {
    "ipaddress": "YOURSERVERIP"
  }  
}
```

### Attribute Hierachy 

node override role, role override cookbook defaults

#### Roles

On a typical Rails project, we might end up with roles similar to the following:
• server - basic configuration to be applied to all servers you manage. For example locking down SSH, ensuring a firewall is enabled, adding public keys for authentication and installing any favourite management tools. This role may also setup your base monitoring configuration
• ruby - installs ruby and provides details of where it should be installed and how. For example rbenv user v system install
• rails-app - any packages generally required by a rails app, for example ImageMagick headers, nodejs for asset compilation etc
• postgres-server - installs postgres and configures authentication methods. Sets up monitoring and alerting for the postgres process.
￼• nginx-server - installs nginx and suitable monitoring

It’s worth noting here that roles can include other roles in a run_list. So it’s possible to create extremely granular roles and then group these together using other roles which are then included in node definitions.

#### A Limitations of Roles
1. One key limitation of roles is that there is no concept of versioning, There is no mechanism in place to ensure that the role we were applying has not changed since the last time we applied it. For this reason, some have gone as far as to describe setting attributes in roles as an antipattern.

### Managing Cookbooks with Berkshelf

Berkshlef provides exactly same functionality as gem in rails.

1. we can define the cookbooks our chef repository is dependent on and the versions of these. We can then use berks install to grab all of these cookbooks and its dependencies.
2.  Generates a .lock file (Berksfile.lock) with the relevant versions for each cookbook.

## Reference:
[https://github.com/ArturT/chef-deploying-rails-applications/tree/master/rdr_redis_example](https://github.com/ArturT/chef-deploying-rails-applications/tree/master/rdr_redis_example "https://github.com/ArturT/chef-deploying-rails-applications/tree/master/rdr_redis_example")
http://dougireton.com/blog/2013/02/16/chef-cookbook-anti-patterns/
http://realityforge.org/code/2012/11/19/role-cookbooks-and-wrapper-cookbooks.html

