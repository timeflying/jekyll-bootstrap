---
layout: post
title: "Ngnix Configuration"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Overview

When a request reaches Nginx, it looks for a virtualhost which defines how and where the request should be routed. Nginx virtualhosts are stored in /etc/nginx/sites-enabled.

##### config/deploy/shared/nginx.conf.erb
```rb
#The upstream block allows us to define a server or group of servers which we can later refer to when using proxy_pass.
#Here we define a single server as unicorn_APP_NAME which points to the unix socket we’ve defined for our unicorn
upstream unicorn_<%= fetch(:full_app_name) %> {
  server unix:/tmp/unicorn.<%= fetch(:full_app_name) %>.sock fail_timeout=0;

server {
#The server_name directive defines the hostname that the virtualhost represents; It can contains wildcards, and list multiple entries on one line wtih seperated by space.
  server_name <%= fetch(:server_name) %>;
  listen 80;
  root <%= fetch(:deploy_to) %>/current/public;
  #The location directive allows us to define configuration parameters which are specific to a particular URL pattern
  location ^~ /assets/ {
    gzip_static on;
    expires max;
    #Adds a flag which is used by proxies to determine whether the file in question is safe to cache.
    add_header Cache-Control public;
}
#This means that it will first look for the existence of the file provided by the url with /index.html appended. If that is not found then it will look for the file specified by the URL, if that is not found that in will pass the request onto the @unicorn location. This means that any valid requests for static assets will never be passed to our Rails app server.
  try_files $uri/index.html $uri @unicorn;
  location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    #forward the request onto our Rails application using the unicorn_full_app_name location we defined at the top of this file.
    proxy_pass http://unicorn_<%= fetch(:full_app_name) %>;
}
  error_page 500 502 503 504 /500.html;
  client_max_body_size 4G;
  keepalive_timeout 10;
}
```

### Forcing HTTPS
To forcing https, add the line `rewrite     ^   https://$server_name$request_uri? permanent;` to server block for port 80

#### Adding SSL
To add an SSL Certificate you will need:

• Your SSL Certificate

• Any intermediate Certificates (see Chaining SSL Certificates below) 

• Your SSL Private Key

To make server support ssl, add fowlling lins to `server` block:

```
listen 443;
ssl on;
ssl_certificate <%= fetch(:deploy_to) %>/shared/ssl_cert.crt;
ssl_certificate_key <%= fetch(:deploy_to) %>/shared/ssl_private_key.key;
```

This enables SSL and sets the location of the SSL certificate and corresponding private Key. These should not be included in source control so you’ll need to SSH into your server and create the relevant files.
Once you’ve added your certificates and run deploy:setup_config again to copy the updated NGinx vitualhost across, you’ll need to restart or reload nginx for the changes to be picked up.

#### Chaining SSL Certificates

Often when purchasing SSL certificates, you’ll be provided with your own certificate, an intermediate certificate and a root certificate. These should all be combined into a single file.

You’ll begin with three files:
  • Your Certificate
  • Primary Intermediate CA
  • Secondary Intermediate CA
These should be combined into a single file in the order:
  1 YOUR CERTIFICATE
  2 SECONDARY INTERMEDIATE CA
  3 PRIMARY INTERMEDIATE CA
You can do this with the cat command:
  `cat my.crt secondary.crt primary.crt > ssl_cert.crt`

Reference:  [http: //superuser.com/questions/347588/how-do-ssl-chains-work](http: //superuser.com/questions/347588/how-do-ssl-chains-work)

#### Updating SSL Certificates

SSL Certificates will, after a pre agreed time period (often one year), expire and need to be replaced with new ones. Updating them is quite a simple process but one it’s easy to get wrong. For this reason I use a simple bash script to automated switching the old cert for the new one and rolling back in case it all goes wrong.

The script assumes you’re starting with 4 files in `/home/deploy/YOUR_APP_DIRECTORY/shared`:

• my.crt - Your SSL Certificate

• secondary.crt - A secondary intermediate certificate

• primary.crt - A root certificate

• ssl_private_key.key.new - The private key used to generate the certificate

The following script should be stored in the same directory as the target for the certificates.

```sh
#!/bin/bash
if [ $# -lt 1 ]
then
	echo "Usage : $0 command"
	echo "Expects: my.crt, secondary.crt, primary.crt, ssl_private_key.key.new"
	echo "Commands:"
	echo "load_new_certs"
	echo "rollback_certs"
	echo "cleanup_certs"
	exit
fi
case "$1" in
	load_new_certs) echo "Copying New Certs"
		cat my.crt secondary.crt primary.crt > ssl_cert.crt.new
		mv ssl_cert.crt ssl_cert.crt.old
		mv ssl_cert.crt.new ssl_cert.crt
		mv ssl_private_key.key ssl_private_key.key.old
		mv ssl_private_key.key.new ssl_private_key.key

		sudo service nginx reload
		;;
	rollback_certs) echo "Rolling Back to Old Certs"
		mv ssl_cert.crt ssl_cert.crt.new
		mv ssl_cert.crt.old ssl_cert.crt
		mv ssl_private_key.key ssl_private_key.key.new
		mv ssl_private_key.key.old ssl_private_key.key

		sudo service nginx reload
		;;
	cleanup_certs) echo "Cleaning Up Temporary Files"
		rm ssl_cert.crt.old
		rm ssl_private_key.key.old
		rm my.crt
		rm secondary.crt
		rm primary.crt
		;;
	*) echo "Command not known"
		;;
esac
```
Execute `./script_name load_new_certs`
to swap in the new certificates and reload nginx. If, after testing the site, something isn’t right,
you can execute:
`./script_name rollback_certs`to revert to the previous ones. And then repeat load_new_certs once you’ve resolved the issue.

gOnce you have the new certificates working as intended, you can use:
`./script_name cleanup_certs`
To remove the temporary and legacy files created.