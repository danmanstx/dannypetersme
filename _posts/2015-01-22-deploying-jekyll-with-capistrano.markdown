---
layout: post
title:  "Deploying Jekyll With Capistrano 3 to a Digital Ocean Droplet"
date:   2015-01-22 19:27:00
comments: true
categories: jekyll capistrano nginx
---

>First post here, but yeah I thought I'd document how I got this blog up and running on a Droplet running Ubuntu 14.04.

#### Guide
* [Blog Set Up](#blog)
* [Server (Droplet) Set Up](#server)
* [Capistrano 3 Set Up](#cap)
* [Deploying](#deploy)


<br>

### <a name="blog">Blog Set Up</a>
--------------------------

To get started with the blog, we are using awesome [jekyll](http://jekyllrb.com)

> now I really like to use [rvm](http://rvm.io) so I will assume that going forward, so if you need help setting it up locally follow that link.

`gem istall jekyll`

Now to get the most basic blog up and going, you can use

{% highlight bash %}
  $ gem install jekyll
~ $ jekyll new blog
~ $ cd blog
~/blog $ jekyll serve
# => Now browse to http://localhost:4000
{% endhighlight %}

and boom! We are now serving the basic jekyll blog locally on port `4000`.

To set up our basic `git` repo for [github](http://github.com) we can use the following commands.

>be sure to change your origin to the repo you create on github

{% highlight bash %}
$ git init
$ git add . # add all files
$ git commit -m "First Commit" # commit message
$ git remote add origin git@github.com:danmanstx/blog.git
$ git push -u origin master
{% endhighlight %}

<br >

### <a name="server">Server (Droplet) Set Up</a>
----------------

To begin, I had a preconfigued droplet with a few rails apps aready on it and rvm [(guide)](https://www.digitalocean.com/community/tutorials/how-to-install-ruby-on-rails-on-ubuntu-14-04-using-rvm). In this case the droplet was the 1gb for $10/month, and this seems more than enough for running 3 rails apps and hosting this blog.

To get multiple users set up and going on your droplet [use this guide](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04). Following along, my new user is named `rails` and I just run the apps straight out of folders in its home directory for convenience. However Common practice is to use something like www


So to start, I created another folder for my blog as the `rails` user, and set up some needed default folders.

{% highlight bash %}
# switch to rails user if you aren't already
$ sudo su - rails
~ $ mkdir blog
~ $ cd blog
~/blog $ mkdir shared
~/blog $ mkdir shared/log
{% endhighlight %}

*Just be sure this user doesn't have root access and only [owns](https://en.wikipedia.org/wiki/Chown) these folders.*

Next, I got NGINX installed.

{% highlight bash %}
sudo apt-get install nginx
{% endhighlight %}

simple enough right? Well, now we need to configure it. I don't believe i changed much in this file `/etc/nginx/nginx.conf`, but here it is.

{% highlight nginx linenos %}

user www-data;
worker_processes 4;
pid /run/nginx.pid;

events {
  worker_connections 768;
  # multi_accept on;
}

http {
  # Basic Setting
  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout 65;
  types_hash_max_size 2048;

  server_names_hash_bucket_size 64;
  # server_name_in_redirect off;

  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  # SSL Setting
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
  ssl_prefer_server_ciphers on;

  # Logging Setting
  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;

  # Gzip Setting
  gzip on;
  gzip_disable "msie6";

  # Virtual Host Config
  include /etc/nginx/conf.d/*.conf;
  include /etc/nginx/sites-enabled/*;
}
{% endhighlight %}

However, I did have to create a site configuration file in `sites-available` using the following command:

`sudo vi /etc/nginx/sites-available/<your_site_name>.conf`

>be sure to note that you probably don't want to use `<your_site_name>`

Next, I just created a basic `server` block that looks at a directory named for my blog in the `rails` user's home directory.

{% highlight nginx linenos %}
server {
  listen 80;
  server_name dannypeters.me www.dannypeters.me;

  access_log /home/rails/blog/shared/log/nginx.access.log;
  error_log /home/rails/blog/shared/log/nginx.access.log;

  location / {
    root /home/rails/blog/current/public/;
    index index.html index.htm;
  }
}
{% endhighlight %}

Now to follow `nginx` convention i symlinked this config file to the `sites-enabled` folder

`ln -s /etc/nginx/sites-available/<your_site_name>.conf /etc/nginx/sites-enabled/<your_site_name>.conf`

<br>

### <a name="cap">Capistrano Set Up</a>
--------------------------
> this is done on your local machine

To begin we need to make sure that Capistrano is installed, if not

`gem install capistrano`

and then set up a `Gemfile` to monitor the gems we will be using.

`~/blog $ vi Gemfile`

{% highlight ruby linenos %}
source 'https://rubygems.org'

ruby '2.1.5'
gem 'jekyll'
gem 'capistrano-rvm'
gem 'capistrano-bundler'

gem 'rvm1-capistrano3', require: false
{% endhighlight %}

Next, we need to capify our blog. Just run the following command from your blog directory.

> with capistrano 2.x this command was `capify`
{% highlight bash %}
$ cap install
{% endhighlight %}

Now we need to edit some of the files it created.

<br>
First, `Capfile`, we need to add `require 'rvm1/capistrano3'` because we are using rvm, and also uncomment some things.

{% highlight ruby linenos %}
# Load DSL and set up stages
require 'capistrano/setup'

# Include default deployment tasks
require 'capistrano/deploy'

require 'capistrano/rvm'
require 'rvm1/capistrano3'
# require 'capistrano/rbenv'
# require 'capistrano/chruby'
require 'capistrano/bundler'
# require 'capistrano/rails/assets'
# require 'capistrano/rails/migrations'
# require 'capistrano/passenger'

# Load custom tasks from `lib/capistrano/tasks' if you have any defined
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }

{% endhighlight %}

<br>
Next, Our `config/production.rb` file, be sure to edit the `server` line with your settings.

{% highlight ruby linenos %}
set :stage, :production

# Extended Server Syntax
# ======================
# This can be used to drop a more detailed server definition into the
# server list. The second argument is a, or duck-types, Hash and is
# used to set extended properties on the server.

server 'xxx.xxx.xxx.xxx', user: 'rails', port: 22, roles: %w{web app}

set :bundle_binstubs, nil

set :bundle_flags, '--deployment --quiet'
set :rvm_type, :user


SSHKit.config.command_map[:rake]  = "bundle exec rake"
SSHKit.config.command_map[:rails] = "bundle exec rails"

namespace :deploy do

  desc "Restart application"
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
      # execute :touch, release_path.join("tmp/restart.txt")
    end
  end

  after :finishing, "deploy:cleanup"

end
{% endhighlight %}

<br>
Finally, We need to set up the `deploy.rb` file.

The key here is this bit of code which builds your jekyll blog into the public folder so you don't need to compile and commit before each deploy. It's my `build_public` task below.

{% highlight ruby linenos %}
# config valid only for Capistrano 3.1
# lock '3.2.1'

set :application, 'blog'
set :repo_url, "git@github.com:danmanstx/#{fetch(:application)}.git"

# Default deploy_to directory is /var/www/my_app
set :deploy_to, "/home/rails/#{fetch(:application)}"

# Default value for :scm is :git
set :scm, :git

# Default value for :log_level is :debug
set :log_level, :debug

# Default value for keep_releases is 5
set :keep_releases, 5

namespace :deploy do

  desc 'Restart application'
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
      # Your restart mechanism here, for example:
      # execute :touch, release_path.join('tmp/restart.txt')
    end
  end

  before :restart, :build_public do
    on roles(:app) do
      within release_path do
        execute '/home/rails/.rvm/gems/ruby-2.1.5/wrappers/jekyll',  "build --destination public"
      end
    end
  end

  after :publishing, :restart

end


{% endhighlight %}

At fist I was getting the following error:

`remote: /usr/bin/env: ruby_executable_hooks: No such file or directory`

So I switched from using the following command calling jekyll directly and ended up using the rvm wrapper, with help from [this stack overflow answer](http://stackoverflow.com/questions/26247926/how-to-solve-usr-bin-env-ruby-executable-hooks-no-such-file-or-directory)

{% highlight ruby linenos %}
execute :jekyll, "build --destination public"`
{% endhighlight %}

became:

{% highlight ruby linenos %}
execute '/home/rails/.rvm/gems/ruby-2.1.5/wrappers/jekyll',  "build --destination public"
{% endhighlight %}

<br>

### <a name="deploy">Deploying</a>
--------------------------


Finally, any new post or updates can be commited and deployed using the following commands.

{% highlight bash %}
$ git add .
$ git commit -m "new post"
$ git push
$ cap production deploy
{% endhighlight %}

<br>

###Congratulations you are now a blogger. 🍻
