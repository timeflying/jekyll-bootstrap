---
layout: post
title: Unicorn configuring
---


## How to Write Custom Capistrano Tasks?
#### Sample: unicorn restart task.
##### lib/capistrano/tasks/restart.cap


	namespace :deploy do
	  desc 'Restart unicorn application' task :restart do
    	on roles(:app), in: :sequence, wait: 5 do
	      sudo "/etc/init.d/unicorn_#{fetch(:full_app_name)} restart"
    	end 
	  end
	end


`on roles(:app), in: :sequence, wait: 5 do` means on every server with `app` role, execute the task sequentially every 5 seconds.

## Unicorn Configuration and Zero Downtime Deployment

##### config/deploy/shared/unicorn.rb.erb 

	root = "<%= current_path %>"
	working_directory root
	pid "#{root}/tmp/pids/unicorn.pid"
	stderr_path "#{root}/log/unicorn.log"
	stdout_path "#{root}/log/unicorn.log"

	listen "/tmp/unicorn.<%= fetch(:full_app_name) %>.sock"
	worker_processes <%= fetch(:unicorn_worker_count) %>
	timeout 40
	...


We set the working directory for Unicorn to be the path of the release, e.g. /home/deploy/APP_NAME/current.

We have a pid file written to the `tmp/pids` sub-directory of our app root. Notice that the `tmp/pids` directory is included in `linked_dirs` in our `deploy.rb` file. This means that our pid is stored in the shared folder and so will persist across deploys. This is particularly important when setting up zero downtime deploys as the contents of current will change but we will still need access to the existing pids.

The `listen` command sets up the unicorn master process to accept connections on a unix socket stored in `/tmp`. A socket is a special type of unix file used for inter process communication. In our case it will be used for allowing Nginx and the Unicorn master process to communicate. 

### Timeout settings
Finally timeout sets the maximum length a request will be allowed to run for before being killed and a 500 error returned. In the sample configuration this is set to 40 seconds. This is generally too generous for a modern web application, typically a value of 15 or below is acceptable. 

In this case the long timeout is because it’s not unusual for apps being put into production for the first time to have some “Rough Edges” with a few requests, often admin ones, taking a long time. A tight timeout value to start with can make getting set up frustrating. Once your app is up and running smoothly, I’d suggest decreasing this based on the longest you’d expect a request to your specific app to take, plus a margin of 25 - 50% for error.

### Note for UNIX sgnals processing of Unicorn
Something to be aware of is that Unicorns use of signals is not entirely standard.

* In standard: 
  the QUIT signal for telling a process to exit immediately   the TERM signal to trigger a graceful shut down.

* Unicorn:
  the QUIT signal to trigger a graceful shut down, 
  the TERM is used to immediately kill all worker processes and then immediately stop the master process.

### Unicorn Init script

The unicorn init script is created when we run cap `deploy:setup_config`. It is stored in `config/deploy/shared/unicorn_init.sh.erb` and symlinked to `/etc/init.d/unicorn_YOUR_APP_NAME`.

You can manually start, stop, force-stop the Unicorn server.

```sh
 /etc/init.d/unicorn_YOUR_APP_NAME start|stop|force-stop
```

### Zero Downtime Deployment

Zero downtime deployment with Unicorn allows us to do the following:
  1.  Deploy New Code
  2.  Start a new Unicorn master process (and associated workers) with the new code, without stopping the existing master
  3. Only once the new master (and associated workers) has loaded, stop the old one and start sending requests to the new one
  
  In our init script, this is taken care of be the restart task:

```sh
sig USR2 && echo reloaded OK && exit 0
  echo >&2 "Couldn't reload, starting '$CMD' instead"
  run "$CMD"
  ;;
```

This sends the USR2 signal which the Unicorn manual states:
> re-execute the running binary. A separate QUIT should be sent to the original process once the child is verified to be up and running.

This takes care of starting the new master process, once the new master has started, we must take care of killing the original (old) master once the new one has started. This is handled by the before_fork block in our Unicorn config file

##### config/deploy/shared/unicorn.rb.erb 

```rb
before_fork do |server, worker|
   defined?(ActiveRecord::Base) and
    ActiveRecord::Base.connection.disconnect!
  # Quit the old unicorn process
  old_pid = "#{server.config[:pid]}.oldbin"
  if File.exists?(old_pid) && server.pid != old_pid
    puts "We've got an old pid and server pid is not the old pid"
    begin
      Process.kill("QUIT", File.read(old_pid).to_i)
      puts "killing master process (good thing tm)"
    rescue Errno::ENOENT, Errno::ESRCH
      puts "unicorn master already killed"
    end
  end
end
```


The block defined above begins by gracefully closing any open ActiveRecord connections. It then checks to see if we have a Pidfile with the .oldbin extension which is automatically created by Unicorn when handling a USR2 restart. If so it sends the “QUIT” signal to the old master process to shut it down gracefully.

The final section of our Unicorn config file (unicorn.rb) defines an after_fork block which is run once by each worker, once the master process finishes forking it:

```rb
after_fork do |server, worker|
  port = 5000 + worker.nr
  child_pid = server.config[:pid].sub('.pid', ".#{port}.pid")
  system("echo #{Process.pid} > #{child_pid}")
   defined?(ActiveRecord::Base) and
    ActiveRecord::Base.establish_connection
end
```

This creates Pidfiles for the forked worker process so that we can monitor it individually with Monit. It also establishes an ActiveRecord connection for the new worker process.

###Gemfile Reloading

There are two very common problems which are complained of when using the many example Unicorn zero downtime configurations available:
1. New or updated gems aren’t loaded, so whenever the Gemfile is changed, the application has to be stopped and started again.
2. Zero downtime fails every fifth or so deploy and the application has to be manually started and stopped. It then works again for five or so deploys and the cycle repeats.

If this is not specified then the Gemfile path from when the master process was first started will be used. Initially it may seem like this is fine, our Gemfile path is always going to be `/home/deploy/apps/APP_NAME/current/Gemfile` 
When deploying with Capistrano, the code is stored in `/deploy/apps/APP_NAME/releases/DATESTAMP`
The Gemfile path which will be used by the Unicorn process is the resolved symlink path, e.g. /deploy/apps/APP_NAME/releases/20140324162017 rather than the current directory.

In deploy.rb we have this:
`set :keep_releases, 5`
to have Capistrano delete all releases except the five most recent ones. This means that if we’re not setting the Gemfile path back to current on every deploy, we’ll eventually delete the release which contains the Gemfile Unicorn is referencing, this will prevent a new master process from starting and you’ll see a
`Gemfile Not Found` Exception in `log/unicorn.log` file.

We avoid the above two problems with the following `before_exec` block in our Unicorn configuration file:

##### config/deploy/shared/unicorn.rb.erb 

	# Force unicorn to look at the Gemfile in the current_path
	# otherwise once we've first started a master process, it
	# will always point to the first one it started.
	before_exec do |server|
	  ENV['BUNDLE_GEMFILE'] = "<%= current_path %>/Gemfile"
	end


`before_exec` is run before Unicorn starts the new master process


