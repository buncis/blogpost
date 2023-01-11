#install devtools
sudo add-apt-repository ppa:chris-lea/redis-server
sudo apt-get update
sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates redis-server redis-tools

#GENERATE NEW SSH dan upload ke github
https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent

#install ruby
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
git clone https://github.com/rbenv/rbenv-vars.git ~/.rbenv/plugins/rbenv-vars
exec $SHELL
rbenv install 3.2.0
rbenv global 3.2.0

#add no doc no ri to gem rc
echo "gem: --no-document" > ~/.gemrc

#install bundler
gem install bundler

#install passenger and nginx
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger focal main > /etc/apt/sources.list.d/passenger.list'
sudo apt-get update
sudo apt-get install -y nginx-extras libnginx-mod-http-passenger
<!-- ini buat enabling passenger module and restart nginx -->
if [ ! -f /etc/nginx/modules-enabled/50-mod-http-passenger.conf ]; then sudo ln -s /usr/share/nginx/modules-available/mod-http-passenger.load /etc/nginx/modules-enabled/50-mod-http-passenger.conf ; fi
sudo ls /etc/nginx/conf.d/mod-http-passenger.conf
<!-- set ruby di mod-http-passenger.conf -->
passenger_ruby /home/deploy/.rbenv/shims/ruby;
sudo service nginx start
<!-- sampai sini seharusnya kalo buka ip static public
  keliatan itu halaman default nginx  -->

#install postgresql 
sudo apt-get install postgresql postgresql-contrib libpq-dev
sudo su - postgres
<!-- bikin user postgresq sesuai user login -->
createuser --pwprompt ubuntu
<!-- bikin database buat appnya -->
createdb -O deploy myapp

#mengarahkan nginx ke ruby app dari passenger
#buang config nginx default
set server config nginx 
server {
  listen 80;
  listen [::]:80;

  server_name _;
  root /home/deploy/myapp/current/public;

  passenger_enabled on;
  passenger_app_env production;

  location /cable {
    passenger_app_group_name myapp_websocket;
    passenger_force_max_concurrent_requests_per_process 0;
  }

  # Allow uploads up to 100MB in size
  client_max_body_size 100m;

  location ~ ^/(assets|packs) {
    expires max;
    gzip_static on;
  }
}

<!-- udah selesai bikin file config baru rload nginx lagi -->
sudo service nginx reload

# SETUP CAPISTRANO BIAR DEPLOY GAMPANG

gem 'capistrano', '~> 3.11'
gem 'capistrano-rails', '~> 1.4'
gem 'capistrano-passenger', '~> 0.2.0'
gem 'capistrano-rbenv', '~> 2.1', '>= 2.1.4'
bundle
cap install STAGES=production

# edit capfile

require 'capistrano/rails'
require 'capistrano/passenger'
require 'capistrano/rbenv'

# edit config/deploy.rb

set :rbenv_type, :user
set :rbenv_ruby, '3.2.0'

set :application, "myapp"
set :repo_url, "git@github.com:username/myapp.git"

# Deploy to the user's home directory
set :deploy_to, "/home/deploy/#{fetch :application}"

append :linked_dirs, 'log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', '.bundle', 'public/system', 'public/uploads'

# Only keep the last 5 releases to save disk space
set :keep_releases, 5

# Optionally, you can symlink your database.yml and/or secrets.yml file from the shared directory during deploy
# This is useful if you don't want to use ENV variables
# append :linked_files, 'config/database.yml', 'config/secrets.yml'

# config/deploy/production.rb
server '1.2.3.4', user: 'ubuntu', roles: %w{app db web}

set :ssh_options, {
      keys: %w(/home/bc/aws/LightsailDefaultKey-ap-southeast-1.pem),
      forward_agent: false,
      auth_methods: %w(publickey),
    }

<!-- sampe sini udah bisa deploy make -->

cap production deploy

tapi ada requirements buat bikin folder and env vars

jadi vim /home/ubuntu/myapp/.rbenv-vars

# For Postgres
DATABASE_URL=postgresql://deploy:PASSWORD@127.0.0.1/myapp

RAILS_MASTER_KEY=ohai
SECRET_KEY_BASE=1234567890

STRIPE_PUBLIC_KEY=x
STRIPE_PRIVATE_KEY=y
# etc...

# SETUP SSL SERVER BUAT DOMAIN
<!-- install certbot -->
sudo snap install --classic certbot
sudo certbot --nginx
sudo certbot renew --dry-run

# setup ulang nginx buat handle SSL

server {

  listen 80;
  listen [::]:80 ipv6only=on;

  listen [::]:443 ssl ipv6only=on; # managed by Certbot
  listen 443 ssl; # managed by Certbot
  ssl_certificate /etc/letsencrypt/live/buncislab.com/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/buncislab.com/privkey.pem; # managed by Certbot
  include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

  server_name _;
  root /home/ubuntu/blogpost/current/public;

  passenger_enabled on;
  passenger_app_env production;
  
  location /cable {
    passenger_app_group_name blogpost_websocket;
    passenger_force_max_concurrent_requests_per_process 0;
  }

  client_max_body_size 100m;

  location ~ ^/(assets|packs) {
    expires max;
    gzip_static on;
  }

}
