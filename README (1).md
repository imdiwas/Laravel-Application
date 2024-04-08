
# Install and Set Up Laravel, Nginx, and MySQL With Docker Compose on Centos 7


## CREATE A NON-ROOT USER
We are going to create a non-root user with sudo privileges.
```bash
  sudo su
  adduser app
  usermod -aG sudo app
  su app
```
## Install Docker 

Older versions of Docker went by docker or docker-engine. Uninstall any such older versions before attempting to install a new version, along with associated dependencies.

```bash
  sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

Set up the repository

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
Install Docker Engine

To install the latest version,
```bash
  sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
To Start the Docker 
```bash
  sudo systemctl start docker
```

To check Status of the Docker 
```bash
  sudo systemctl status docker
```
You have now successfully installed and started Docker Engine.

## Install Docker-compose
Install Curl Command.

```bash
  sudo yum install curl
```
Download Docker compose.
```bash
  sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
Change the file permissions to make the software executable.
```bash
  sudo chmod +x /usr/local/bin/docker-compose
```
Docker Compose does not require running an installation script. As soon as you download the software, it will be ready to use.

To verify the installation
```bash
  docker–compose –-version
```

## CLONING THE LARAVEL REPO

You need to check that you are in the home directory to clone laravel
```bash
 cd ~
 git clone https://github.com/laravel/laravel.git laravel-app
```
Move into the laravel-app directory. 
```bash
  cd ~/laravel-app

```
Set permissions on the project directory so that it is owned by your non-root user
```bash
  sudo chown -R app:app ~/laravel-app

```

## CREATING DOCKER COMPOSE & DOCKERFILE
You will write a docker-compose file that defines your web server, database, and application services.

```bash
  version: '3'
services:
  
  #PHP Service
  app:
    build:
      args:
        user: app
        uid: 1000
      context: .
      dockerfile: Dockerfile
    image: app/php
    container_name: app
    restart: unless-stopped
    tty: true
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    working_dir: /var/www
    volumes:
      - ./:/var/www
      - ./php/local.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      - app-network

  #Nginx Service
  webserver:
    image: nginx:alpine
    container_name: webserver
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./:/var/www
      - ./nginx/conf.d/:/etc/nginx/conf.d/
      - ./nginx-logs:/var/log/nginx
    networks:
      - app-network

  #MySQL Service
  db:
    image: mysql:5.7.22
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_USER: ${DB_USERNAME}
    volumes:
      - dbdata:/var/lib/mysql/
      - ./mysql/my.cnf:/etc/mysql/my.cnf
    networks:
      - app-network

#Docker Networks
networks:
  app-network:
    driver: bridge
#Volumes
volumes:
  dbdata:
    driver: local
```

## DOCKERFILE
Your Dockerfile will be located in your ~/laravel-app directory. Create the file.
```bash
  nano ~/laravel-app/Dockerfile

```
Dockerfile

```bash
 FROM php:8.0.2-fpm

# Arguments defined in docker-compose.yml
ARG user
ARG uid

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Create system user to run Composer and Artisan Commands
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user

# Set working directory
WORKDIR /var/www

USER $user

```
## CONFIGURING PHP, NGINX & MYSQL.

```bash
  nano ~/laravel-app/Dockerfile

```
## PHP

You can set up the PHP service to operate as a PHP processor for incoming requests from Nginx now that your infrastructure has been established in the docker-compose file.

The local.ini file must be created inside the php folder in order to configure PHP.
```bash
  mkdir ~/laravel-app/php
  nano ~/laravel-app/php/local.ini

```
You’ll add the following code to set size restrictions for uploaded files as an example of how to setup PHP.
```bash
  
upload_max_filesize=40M
post_max_size=40M

```
Save the file, then close your editor.

After creating your PHP local.ini file, you may proceed to set up Nginx.

## Nginx

Create the app.conf configuration file
```bash
  mkdir -p ~/laravel-app/nginx/conf.d
  nano ~/laravel-app/nginx/conf.d/app.conf

```
Add the following code to the file to specify your Nginx configuration:
```bash
  server {
    listen 80;
    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/public;
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}

```
Save the file and close your editor.
## MySQL
To configure MySQL, you will create the my.cnf file in the mysql folder.
```bash
  mkdir ~/laravel-app/mysql
  nano ~/laravel-app/mysql/my.cnf

```
In the file, add the following code to enable the query log and set the log file location

```bash
[mysqld]
general_log = 1
general_log_file = /var/lib/mysql/general.log

```
## Running the Containers and Changing Environment Parameters. 
Making a copy of the.env.example file that Laravel includes by default and renaming it to.env — the file Laravel expects to describe its environment.
```bash
  cp .env.example .env
  nano .env

```
Locate the block that defines DB_CONNECTION and make the necessary changes to match your configuration
```bash
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=app
DB_PASSWORD=your_laravel_db_password

```
## Running the Application with Docker Compose.
To create the application image and launch the services we specified in our configuration, we’ll now utilize docker-compose instructions.

Use the command below to create the app image.
```bash
docker-compose build app


```
When the build is finished, you can run the environment in background mode with.
 
```bash 
docker-compose up -d
```
To show information about the state of your active services.
```bash 
docker-compose ps 
```
To run tasks in the service containers, such as a ls -l to display comprehensive information about files in the application directory, use the docker-compose exec command.

```bash 
docker-compose exec app ls -l 
```
Check if you see anything like composer.lock in the outputs. If it is there we run this command to remove it.

```bash 
docker-compose exec app rm -rf vendor composer.lock 
```
We’ll now run composer install to install the application dependencies.

```bash 
docker-compose exec app composer install 
```
Before testing the application, the last step is to create a special application key using the artisan Laravel command-line tool.

```bash 
docker-compose exec app php artisan key:generate
```
As a final step, visit http://your_server_ip in the browser. 

##   
##  
## Setup a Vue.Js application and deploy using Docker.

You have to write a Dockerfile to deoply vue.js application on Docker. 

```bash 
# Dockerfile

# Use the official Node.js 14 image as the base image
FROM node:14

# Set the working directory to /vue-app
WORKDIR /vue-app

# Copy the package.json and package-lock.json files to the container
COPY package*.json ./

# Install the dependencies
RUN npm install

# Copy the rest of the application code to the container
COPY . .

# Expose the port that the application will run on
EXPOSE 8080

# Start the application
CMD ["npm", "start"]
```
To build and run the Docker container, you can use the following commands.
```bash
# Build the Docker image
docker build -t my-vue-app .

# Run the Docker container
docker run -p 8080:8080 -d my-vue-app
```