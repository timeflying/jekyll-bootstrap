---
layout: post
title: "upgrade ruby to 2.3.0, nginx to 1.6.2, Passenger to 4.0.58"
description: ""
category:
tags: [nginx, passenger, ruby, deploy]
---

##Upgrade ruby##

1. download source code : https://www.ruby-lang.org/en/news/2015/12/25/ruby-2-3-0-released/
   and untar it to /tmp folder
2. cd /tmp/ruby-2.3.0
3. ./configure --prefix=/opt/ruby-2.3.0  # install ruby to /opt/ruby-2.3.0
4. make
5. make install
6. mv /usr/local/bin/ruby /usr/local/bin/ruby-2.1.8
7. ln -s /opt/ruby-2.3.0/bin/ruby /usr/local/bin/ruby  # use ruby-2.3.0 as default ruby

##Upgrade Passenger##

Because there will be an error after upgrading ruby to 2.3.0

error : https://github.com/phusion/passenger/issues/1362
solve : upgrade passenger to 4.0.58

upgrade passenger

1. gem install passenger -v 4.0.58

##Upgrade nginx##

After upgrading, we have to upgrade nginx also.

1. run : /opt/ruby-2.3.0/lib/ruby/gems/2.3.0/gems/passenger-4.0.58/bin/passenger-install-nginx-module
2. install Nginx to /opt/nginx-1.6.2
3. after install, cp /opt/nginx/conf/nginx.conf to /opt/nginx-1.6.2/conf/nginx.conf
4. in nginx.conf file, change passenger_root value to /opt/ruby-2.3.0/lib/ruby/gems/2.3.0/gems/passenger-4.0.58
