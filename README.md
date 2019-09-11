
# Install the dependencies on the server

All of the following is done in you AWS instance.

Make sure your instance allows inbound and outbound HTTP and HTTPS.

Also make sure your public ssh key is authorized on the server.

On your machine:
```
ssh -i <your-key.pem> ubuntu@<public-ip-of-instance> "echo \"`cat ~/.ssh/id_rsa.pub`\" >> .ssh/authorized_keys"
```
This should add your public key to the authorized_keys on the AWS instance.

## 1. Install RVM

```
gpg --keyserver hkp://pool.sks-keyservers.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB

curl -sSL https://get.rvm.io | bash -s stable
```

If you don't have curl, follow the insttructions in your terminal to install it.

<br>

## 2. Install Ruby

```
rvm install ruby-2.6.3

ruby -v
```

This should output the version of ruby. In this case: 2.6.3

<br>

## 3. Install Rails and Bundler

```
gem install rails -v 6.0.0.rc2

rails -v
```

This should output the version of rails. In this case: 6.0.0.rc2

```
gem install bundler -v 2.0.2

bundler -v
```

This should output the version of bundler. In this case: 2.0.2

<br>

## 3. Install Nginx

```
sudo apt update && sudo apt upgrade

sudo apt install nginx -y

nginx -v
```

This should output the version of Nginx.

<br>

## 4. Install MySQL (and optionally PostgreSQL)

```
sudo apt update && sudo apt upgrade

sudo apt install libmysqlclient-dev -y

sudo apt install postgresql postgresql-contrib -y
```

## 5. Install Yarn and NodeJS

```
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -

echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list

sudo apt update && sudo apt install --no-install-recommends yarn -y

sudo apt install npm -y

sudo apt install nodejs -y
```

These commands should install node, npm and yarn.


```
yarn -v
```

This should output the version of yarn.

```
node -v
```

This should output the version of node.

```
npm -v
```

This should output the version of npm (node package manager).

<br>

# Deploy the project

The following is done mainly on your machine.

## 1. Initial deploy

In your project, change the ip in `config/deploy/production.rb` to your AWS instance's ip.

```
server "<your-ip-here>", user: "ubuntu", roles: %w{app db web}

app = ENV['APP']
if app.nil? or app.empty?
  app = "<name-your-app>"
end

set :application, app
set :rails_env, "development"
set :bundle_without, "production"
set :deploy_to, "/home/ubuntu/apps/#{app}"
set :linked_dirs, %w{tmp/pids tmp/sockets log}
set :linked_files, %w{config/database.yml config/master.key}

role :app, %w{ubuntu@<your-ip-here>}
role :web, %w{ubuntu@<your-ip-here>}
role :db,  %w{ubuntu@<your-ip-here>}
```

<br>

This deploy will be creating directories and will generate an error.

Don't worry, this is normal.

```
cap production deploy
```

This command should stop with an error that says `can't find database.yml`

In your AWS instance:

1. Create a file named `database.yml` in your shared folder
```
nano ~/apps/<your-app-name>/shared/config/database.yml
```
2. Paste this config
```
development:
  host: <your-database-address>
  port: 3306
  adapter: mysql2
  encoding: utf8
  database: <name-of-your-database>
  username: <user-in-database>
  password: <password-in-database>
```
3. In `config/deploy/production.rb` in your project, there's a line where you can find all files to put in `shared`.

```
server "<your-ip-here>", user: "ubuntu", roles: %w{app db web}
app = ENV['APP']
if app.nil? or app.empty?
  app = "<your-app-name>"
end
set :application, app
set :rails_env, "development"
set :bundle_without, "production"
set :deploy_to, "/home/ubuntu/apps/#{app}"
set :linked_dirs, %w{tmp/pids tmp/sockets log}
set :linked_files, %w{config/database.yml config/master.key} **HERE

role :app, %w{ubuntu@<your-ip-here>}
role :web, %w{ubuntu@<your-ip-here>}
role :db,  %w{ubuntu@<your-ip-here>}
```

If there are files in your project that shouldn't go on GitHub, put them here and add them in your shared file like we did for `database.yml`

<br>

## 2. Deploy again

```
cap production deploy
```
This command should finish without problem.

If you get stuck in `Compiling...` ,you might need to allocate some swap. To do so, follow this [link](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-18-04).

<br>

## 3. Configure Nginx and Puma

```
cap production puma:config_nginx
```

On your AWS instance:

```
sudo nano /etc/nginx/sites-avalaible/<your-app-name>_production
```

Change `localhost` to your domain:
```
server {
  listen 80;
  server_name www.<your-domain> <your-domain> <your-app-name>.local;
  root /home/ubuntu/apps/<your-app-name>/current/public;
  try_files $uri/index.html $uri @puma_<your-app-name>_production;

  client_max_body_size 4G;
  keepalive_timeout 10;

  error_page 500 502 504 /500.html;
  error_page 503 @503;

  location @puma_<your-app-name>_production {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_redirect off;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header X-Forwarded-Proto http;
    proxy_pass http://puma_<your-app-name>_production;
    # limit_req zone=one;
    access_log /home/ubuntu/apps/<your-app-name>/shared/log/nginx.access.log;
    error_log /home/ubuntu/apps/<your-app-name>/shared/log/nginx.error.log;
  }

 location ^~ /assets/ {
   gzip_static on;
   expires max;
   add_header Cache-Control public;
 }
```

and while you are there, comment out this block of code:

```
location ^~ /assets/ {
  gzip_static on;
  expires max;
  add_header Cache-Control public;
}
```

```
sudo service nginx restart
```

<br>

## 4. Last deploy

```
cap production deploy
```

Again, if you get stuck with `Compiling...` try and [add swap](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-18-04).

If everything is done correctly, your web-site sould be available online!
