# Guideline to deploy Rails app on cloud server
1. AWS
2. Heroku
# AWS
#### Pre-requisite on your local machine 
1. ruby 2.2.3 or 2.2.1 and rails 4. (using rvm) installed
2. Git installed
3. Nodejs installed
4. bundler installed
5. AWS EC2 instance created
6. Puma configured (Follow the steps from the file Deploy on Heroku Rails+ Postgresql with Puma.)
#### Configuring Puma & Capistrano
Include thesse in GEM file
```
gem 'figaro' # for env variables 
gem 'puma'
group :development do
  gem 'capistrano'
  gem 'capistrano3-puma'
  gem 'capistrano-rails', require: false
  gem 'capistrano-bundler', require: false
  gem 'capistrano-rvm'
end
```
Run 
```
cap install STAGES=production
```
#### Edit deploy.rb in config/deploy.rb
```
lock '3.5.0'
set :application, 'contactbook'
set :repo_url, 'git@github.com:devdatta/contactbook.git' # Edit this to match your repository
set :ssh_options, { forward_agent: true }
set :branch, :master
set :deploy_to, '/home/deploy/contactbook'
set :pty, true
set :linked_files, %w{config/database.yml config/application.yml}
set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system public/uploads public/assets}
set :keep_releases, 5
set :rvm_type, :user
set :rvm_ruby_version, '2.2.3' # Edit this if you are using MRI Ruby
set :puma_rackup, -> { File.join(current_path, 'config.ru') }
set :puma_state, "#{shared_path}/tmp/pids/puma.state"
set :puma_pid, "#{shared_path}/tmp/pids/puma.pid"
set :puma_bind, "unix://#{shared_path}/tmp/sockets/puma.sock"    #accept array for multi-bind
set :puma_conf, "#{shared_path}/puma.rb"
set :puma_access_log, "#{shared_path}/log/puma_error.log"
set :puma_error_log, "#{shared_path}/log/puma_access.log"
set :puma_role, :app
set :puma_env, fetch(:rack_env, fetch(:rails_env, 'production'))
set :puma_threads, [0, 8]
set :puma_workers, 0
set :puma_worker_timeout, nil
set :puma_init_active_record, true
set :puma_preload_app, false

#In config/deploy.rb under namespace :deploy do line add this for assets precompile in capistrano 

desc 'run rake assets precompile task'
  task :assets_precompile do
    on roles(:app) do
      within "#{release_path}" do
        with rails_env: "#{fetch(:rails_env)}"  do
          execute :rake, 'assets:precompile'
        end
      end
    end
  end
end
 ```

#### Login to your server, create a user for the app
```
ssh -i your_ec2_key.pem adminuser@yourserver.com
```
Replace adminuser with the name of an account with administrator privileges or sudo privileges. This is usually admin, ec2-user, root or ubuntu.

Now that you have logged in, you should create an operating system user account for your app. For security reasons, it is a good idea to run each app under its own user account, in order to limit the damage that security vulnerabilities in the app can do.

You should give the user account the same name as your app. But for demonstration purposes, this tutorial names the user account myappuser.
```
$ sudo adduser myappuser
```
We also ensure that that user has your SSH key installed:
```
sudo mkdir -p ~myappuser/.ssh
touch $HOME/.ssh/authorized_keys
sudo sh -c "cat $HOME/.ssh/authorized_keys >> ~myappuser/.ssh/authorized_keys"
sudo chown -R myappuser: ~myappuser/.ssh
sudo chmod 700 ~myappuser/.ssh
sudo sh -c "chmod 600 ~myappuser/.ssh/*"
```
#### Install Git on the server

#### Let EC-2 instance able to access github
1. login to the myappuser that was just created
```
$su - myappuser
```
2. gen ssh key by typing 
```
$ssh-keygen
```
3. copy key & paste to your github
```
$ cat .ssh/id_rsa.pub
```

#### Set authorized_keys
(Capistrano will connect to the EC2 instance via ssh for deployment)
```
$ cat ~/.ssh/id_rsa.pub(@local)
```
copy key & paste to Ec2 instance
```
$ nano .ssh/authorized_keys
```
#### Create some file that Capistrano deploy need
```
$ mkdir <app-name>
$ mkdir -p <app-name>/shared/config
$ nano <app-name>/shared/config/database.yml
```

#### Inside ```<app-name>/shared/config/database.yml``` 
```
production:
  adapter: postgresql
  encoding: unicode
  database: contactbook_production       #edit it to fit your app
  username: deploy                       #edit it to fit your app name on remote postgresql
  password: 123456                       #edit it to fit your app password on remote postgresql
  host: localhost
  port: 5432
```
#### Create application.yml
```
$ nano contactbook/shared/config/application.yml
```
Add all the secret keys here such as api_keys etc. if you are using figaro gem.

# Heroku(using Rails 5 as Api and React as client side framework)
#### Add a package.json file in the root directory of the app and enter 
```
{
  "name": "react-and-rails",
  "engines": {
    "node": "6.3.1"
  },
  "scripts": {
    "build": "cd client && npm install && npm run build && cd ..",
    "deploy": "cp -a client/build/. public/",
    "postinstall": "npm run build && npm run deploy && echo 'Client built!'"
  }
}
```
#### Start Heroku
```
$ heroku create <app-name>
```
#### Verify remote 
```
$ git remote -v
```
#### Tell Heroku to start by building the node app using package.json, and then build the rails app
```
heroku buildpacks:add heroku/nodejs --index 1
heroku buildpacks:add heroku/ruby --index 2
```
#### Add a Procfile in the root directory of the app and tell heroku how to start the rails app
```
web: bundle exec rails s
```
#### Deploy 
```
$ git add .
$ git commit -m "ready for first push to heroku"
$ git push heroku master
```
#### Migrate your database on server
```
$ heroku run rake db:migrate
```
