---
layout: post
title: Rails application deploy by Capistrano
---

1. Add gem

    Gemfile: 

    -----------------------

        gem 'capistrano', '~> 3.3.5' 2
        # rails specific capistrano functions
        gem 'capistrano-rails', '~> 1.1.2'

        # integrate bundler with capistrano
        gem 'capistrano-bundler'

        # if you are using Rbenv
        gem 'capistrano-rbenv', "~> 2.0.3"

2. Install gems:

    `bundle install`
3. Initialize capistrano:
    `bundle exec cap install`
4. Include capistrano tasks:
    
    Capfile

    ----------------------
  
        # Load DSL and Setup Up Stages
        require 'capistrano/setup'
        
        # Includes default deployment tasks
        require 'capistrano/deploy'
        
        # Includes tasks from other gems included in your Gemfile
        #
        # For documentation on these, see for example:
        #
        #   https://github.com/capistrano/rvm
        #   https://github.com/capistrano/rbenv
        #   https://github.com/capistrano/chruby
        #   https://github.com/capistrano/bundler
        #   https://github.com/capistrano/rails/tree/master/assets
        #   https://github.com/capistrano/rails/tree/master/migrations
        #
        # require 'capistrano/rvm'
        require 'capistrano/rbenv'
        # require 'capistrano/chruby'
        require 'capistrano/bundler'
        # require "sidekiq/capistrano"
        #require 'capistrano/rails/assets'
        require 'capistrano/rails/migrations'
        
        # Loads custom tasks from `lib/capistrano/tasks' if you have any defined.
        Dir.glob('lib/capistrano/tasks/*.cap').each { |r| import r }
        Dir.glob('lib/capistrano/**/*.rb').each { |r| import r }
        
    When the Capfile requires capistrano/setup, this:
    • Iterates over the stages defined in config/deploy/
    • For each stage, loads the configuration defined in config/deploy.rb
    • For each stage, loads the stage specific configuration defined in config/deploy/stage_name.rb

5. Updating deploy.rb
  
    The file config/deploy.rb contains the parts of our deployment configuration which applies to all stages. 
    
    config/deploy.rb
    
    -----------------------
      
        # what specs should be run before deployment is allowed to
        # continue, see lib/capistrano/tasks/run_tests.cap
        set :tests, []
        
        # which config files should be copied by deploy:setup_config
        # see documentation in lib/capistrano/tasks/setup_config.cap
        # for details of operations
        set(:config_files, %w(
          nginx.conf
          database.example.yml
          log_rotation
          monit
          unicorn.rb
          unicorn_init.sh
        ))
        
        # which config files should be made executable after copying
        # by deploy:setup_config
        set(:executable_config_files, %w(
          unicorn_init.sh
        ))
        
        # files which need to be symlinked to other parts of the
        # filesystem. For example nginx virtualhosts, log rotation
        # init scripts etc. The full_app_name variable isn't
        # available at this point so we use a custom template {{}}
        # tag and then add it at run time.
        set(:symlinks, [
          {
            source: "nginx.conf",
            link: "/etc/nginx/sites-enabled/{{full_app_name}}"
          },
          {
            source: "unicorn_init.sh",
            link: "/etc/init.d/unicorn_{{full_app_name}}"
          },
          {
            source: "log_rotation",
           link: "/etc/logrotate.d/{{full_app_name}}"
          },
          {
            source: "monit",
            link: "/etc/monit/conf.d/{{full_app_name}}.conf"
          }
        ])
        
        # this:
        # http://www.capistranorb.com/documentation/getting-started/flow/
        # is worth reading for a quick overview of what tasks are called
        # and when for `cap stage deploy`
        
        namespace :deploy do
          # make sure we're deploying what we think we're deploying
          before :deploy, "deploy:check_revision"
          # only allow a deploy with passing tests to deployed
          before :deploy, "deploy:run_tests"
          # compile assets locally then rsync
          after 'deploy:symlink:shared', 'deploy:compile_assets_locally'
          after :finishing, 'deploy:cleanup'
        
          # remove the default nginx configuration as it will tend
          # to conflict with our configs.
          before 'deploy:setup_config', 'nginx:remove_default_vhost'
        
          # reload nginx to it will pick up any modified vhosts from
          # setup_config
          after 'deploy:setup_config', 'nginx:reload'
        
          # Restart monit so it will pick up any monit configurations
          # we've added
          after 'deploy:setup_config', 'monit:restart'
        
          # As of Capistrano 3.1, the `deploy:restart` task is not called
          # automatically.
          after 'deploy:publishing', 'deploy:restart'
        end
        
      When setting variables which are to be used across Capistrano tasks we use the   `set` and `fetch` methods provided by Capistrano. Internally we’re setting and   retrieving values in a hash maintained by Capistrano
      
      The values need to be replaced with your own:
      * branch
      * server_name: a space separated list of hostnames which the website will be       served on. This is used for generating the nginx virtual host.
      * rails_env: the RAILS_ENV the application should run in
      * the server line: replacing example.com with the hostname or IP of the remote   server

6. Setting production stage
  
    confit/deploy/production.rb
    
    ----------------------
    
        set :stage, :production
        set :branch, "master"
        
        # This is used in the Nginx VirtualHost to specify which domains
        # the app should appear on. If you don't yet have DNS setup, you'll
        # need to create entries in your local Hosts file for testing.
        set :server_name, "www.example.com example.com"
        
        # used in case we're deploying multiple versions of the same
        # app side by side. Also provides quick sanity checks when looking
        # at filepaths
        set :full_app_name, "#{fetch(:application)}_#{fetch(:stage)}"
        
        server 'example.com', user: 'deploy', roles: %w{web app db}, primary: true
        
        set :deploy_to, "/home/#{fetch(:deploy_user)}/apps/#{fetch(:full_app_name)}"
        
        # dont try and infer something as important as environment from
        # stage name.
        set :rails_env, :production
        
        # number of unicorn workers, this will be reflected in
        # the unicorn.rb and the monit configs
        set :unicorn_worker_count, 5
        
        # whether we're using ssl or not, used for building nginx
        # config file
        set :enable_ssl, false


7. Generating Remote Configuration Files
  
    Capistrano uses a folder called `shared` to manage files and directories that should persist across releases.
    
    ex: link config/database.yml to shared/config/database.yml
    
    config/deploy.rb
    
    --------------------------
      
        # files we want symlinking to specific entries in shared.
        set :linked_files, %w{config/database.yml}
        ```
        ####How to create initial files?
        Use `deploy:setup_config` task
        This custom task is defined in `lib/capistrano/tasks/setup_config.cap` and configured in this section of `config/deploy.rb`:
        #####config/deploy.rb
        ---------------------------
        ```ruby
        # which config files should be copied by deploy:setup_config
        # see documentation in lib/capistrano/tasks/setup_config.cap
        # for details of operations
        set(:config_files, %w(
          nginx.conf
          database.example.yml
          log_rotation
          monit
          unicorn.rb
          unicorn_init.sh
        ))
      
    When this task is run, for each of the files defined in `:config_files` it will first look for a corresponding .erb file (so for `nginx.conf` it would look for `nginx.conf.erb`) in `config/deploy/#{application}_#{rails_env}/`. If it were not found there it would look for it in `config/deploy/shared/`. Once it finds the correct source file, it will parse the erb and then copy the result to the config directory in your remote shared path.

    config/deploy.rb
  
    ---------------------------
  
        # remove the default nginx configuration as it will tend
        # to conflict with our configs.
        before 'deploy:setup_config', 'nginx:remove_default_vhost'
      
        # reload nginx to it will pick up any modified vhosts from
        # setup_config
        after 'deploy:setup_config', 'nginx:reload'
      
        # Restart monit so it will pick up any monit configurations
        # we've added
        after 'deploy:setup_config', 'monit:restart'
  
    Means that during deploy:setup_config is run, we:
    • Delete the default nginx Virtualhost before deploy:setup_config to stop it over-riding our custom VirtualHost 
    • Reload Nginx to pickup any changes to the VirtualHost
    • Reload Monit to pickup any changes to the Monit configuration
    
8. Managing non-Rails configuration

    The configuration files that need to be managed by the deployment process include:
    • Unicorn Monit definitions
    • Nginx Virtual hosts entries
    • Init scripts for unicorn and any background workers 
    • Log rotation definitions
        
    The below section outlines these symlinks which are created by the `deploy:setup_config` tasks defined in `lib/capistrano/tasks/setup_config.cap`:
        
        set(:symlinks, [
          {
            source: "nginx.conf",
            link: "/etc/nginx/sites-enabled/{{full_app_name}}"
          },
          {
            source: "unicorn_init.sh",
            link: "/etc/init.d/unicorn_{{full_app_name}}"
          },
          {
            source: "log_rotation",
           link: "/etc/logrotate.d/{{full_app_name}}"
          },
          {
            source: "monit",
            link: "/etc/monit/conf.d/{{full_app_name}}.conf"
          }
        ])
        
9. Database
  The database example yml file intentionally doesn’t include actual credentials as these should not be stored in version control.
  Therefore, you need to SSH into your remote server and cd into shared/config, then create and set a database.yml.
10. Deploying 
  `bundle exec cap production deploy:setup_config`
  `cap production deploy`

## References:
[https://github.com/TalkingQuickly/capistrano-3-rails-template](https://github.com/TalkingQuickly/capistrano-3-rails-template)
