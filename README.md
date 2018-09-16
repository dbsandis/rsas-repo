# rsas-repo
Residents Services Administration System
https://circleci.com/blog/continuous-drupal-p1-maintaining-with-docker-git-composer/

Software Used The following software and tools are needed:

    Apache (available in image)
    Composer (available in image)
    Docker (v1.13.0 or newer)
    Docker Compose (v1.10.0 or newer)
    Drupal 8 (installed via Composer)
    Drush (installed via Composer)
    Git (v2.10 or newer)
    MariaDB (or MySQL, a version supported by Drupal 8 via Docker)
    PHP7 (available in image, technically PHP5 could be used, but why when PHP7 is SO MUCH BETTER)

Setting Up The Project Directory
We’re going to create a Drupal-based site called “Residents Services Administration System”. To start, we need to create a root directory to hold our entire project. This directory will be versioned with Git. 


    mkdir -p ~/dbsandis/rsas/
    cd ~/dbsandis/rsas/

Create The Dockerfile
Now there’s a few things we need to create before we can start installing Drupal. The first is a Dockerfile that will contain our actual Drupal website. You can copy & paste the following Dockerfile into ./Dockerfile, which is the root of our project.

      FROM drupal:8.4-apache  
      RUN apt-get update && apt-get install -y \ 	
              curl \ 	
              git \ 	
              mysql-client \ 	
              vim \ 	
              wget  
      RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" && \ 	
              php composer-setup.php && \ 	
              mv composer.phar /usr/local/bin/composer && \ 	
              php -r "unlink('composer-setup.php');"  
      RUN wget -O drush.phar https://github.com/drush-ops/drush-launcher/releases/download/0.4.2/drush.phar && \ 	
              chmod +x drush.phar && \ 	
              mv drush.phar /usr/local/bin/drush  
      RUN rm -rf /var/www/html/*  COPY apache-drupal.conf /etc/apache2/sites-enabled/000-default.conf  WORKDIR /app 


Walking Through The Dockerfile

      FROM drupal:8.4-apache 

We starting off from the official Docker Library Drupal image. This means we don’t need to do the work of setting up Apache and PHP ourselves. More specifically, we don’t have to go about tracking down and installing the specific PHP plugins that Drupal needs. It’s important to use a tag for a recent version of Drupal but it doesn’t need to specifically be the version of Drupal you want to run. This is because we’re not going to use the Drupal files provided in the image, but from Composer instead (later in this post).

      RUN apt-get update && apt-get install -y \ 	
                curl \ 	
                git \ 	mysql-client \ 	vim \ 	wget 

We’re installing some tools we need inside the image for the next steps. mysql-client seems like a weird choice but Drush won’t connect properly to our site’s database without it. Tip, we create the command like this with a multi-line, single RUN step as it improves Docker layer caching.

      RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" && \ 	
                php composer-setup.php && \ 	
                mv composer.phar /usr/local/bin/composer && \ 	
                php -r "unlink('composer-setup.php');" 
                
Here we install Composer, the PHP dependency manager. Everything that’s not already installed by you or the Docker image will be handled by Composer. Composer is installed for the root user. You’ll see a warning over and over that you shouldn’t use the root user with Composer. We’re working in a Docker image where the only regular user is the root user. So, despite the warnings, this is okay in that scenario. On your own computer, don’t use the root user.

      RUN wget -O drush.phar https://github.com/drush-ops/drush-launcher/releases/download/0.4.2/drush.phar && \ 	
                chmod +x drush.phar && \ 	
                mv drush.phar /usr/local/bin/drush 

We’re installing Drush Launcher. Makes using Drush on the command-line easier now that it’s no longer recommended to install Drush globally.

      RUN rm -rf /var/www/html/* 
      
This deletes the copy of Drupal that ships with the Docker image. You can use this if you wanted, but in this post we’re installing and tracking Drupal with Composer instead with the help of Git.

      COPY apache-drupal.conf /etc/apache2/sites-enabled/000-default.conf 
      
Copy over the custom Apache VirtualHost configuration file to tell Apache where we want to host our website within the filesystem.
Create Apache VHost Config
That Apache VirtualHost config file we just mentioned, let’s create it. This file should be located at ./apache-drupal.conf.

    <VirtualHost *:80> 	
      ServerAdmin webmaster@localhost 	
      DocumentRoot /app/web  	
      
      <Directory /app/web> 		
        AllowOverride All 		
        Require all granted 	
       </Directory>  
       
       ErrorLog ${APACHE_LOG_DIR}/error.log 	
       CustomLog ${APACHE_LOG_DIR}/access.log 
       combined  
     </VirtualHost>
     # vim: syntax=apache ts=4 sw=4 sts=4 sr noet 
     
Create The Docker Compose File
Our Drupal site is going to be composed of two Docker images. The main image which holds the Drupal specific stuff, which we designated with the Dockerfile we made, and a database image. Docker Compose will allow use to create containers from these two images and have them connected to each other so that they “just work”.
The Docker Compose file should also be created at the root of our project, ./docker-compose.yml.

      version: '3' 
      services:
        db:     
          image: mariadb:10.2     
          environment:       
            MYSQL_DATABASE: drupal       
            MYSQL_ROOT_PASSWORD: xxxxxxxxxx     
          volumes:       
            - db_data:/var/lib/mysql     
            restart: always   
         drupal:     
            depends_on:       
              - db     
            build: .     
            ports:       
              - "8080:80"     
            volumes:       
              - ./app:/app     
            restart: always 
         volumes:   db_data: 

Walking Through The Docker Compose File

    version: '3' 

Many blog posts around the Internet with example Compose files will use version 2. We are using newer features of Docker Compose and thus this needs to be version 3 or higher.

    services:   
      db:     
        image: mariadb:10.2     
        environment:       
            MYSQL_DATABASE: drupal       
            MYSQL_ROOT_PASSWORD: xxxxxxxxxxxxxx
            
We create a container here called “db” that is made from the MariaDB v10.2 Docker image. MySQL and Postgress could be used instead, anything that Drupal supports. We also tell MariaDB to create a new database called “drupal” on startup and set the MariaDB root user’s password to “WaterfallSucks”. Feel free to use whichever DB name and password you like. The Docker Hub page for MariaDB provides more information on available environment variables.
        
        volumes:       
          - db_data:/var/lib/mysql 
          
We specify that we want to use a Docker Volume called db_data to store the contents of /var/lib/mysql. This directory is where MariaDB stores its information for the databases it contains. It allows us to shutdown the containers and start them again later without losing any data saved in the database.
      
        drupal:     
          depends_on:     - db 
          
We specify a second container called Drupal. The depends_on key tells Docker Compose to not start this container until the db container has started. It’s best not to start Drupal because the database exists.
      
        build: . 

Instead of using image like we did for the DB and then specifying a Docker image, this tells Docker Compose to use the local Dockerfile as the basis for our image. This means we can build on the fly without running a separate command, or worse, pushing to a public Docker repository for a quick dev change.

        ports:       - "8080:80" 

We map port 8080 on our local machine to port 80 within the Docker container. This allows us to visit http://localhost:8080 in our browser and see our running site within the container. Port 8080 can be any available port you have locally.
    
        volumes:       - ./app:/app 

We’re creating a bind mount here instead of a traditional volume. This takes the ./app directory that we will have in our repository on our local machine and makes it available as /app with the Docker container. This is how we provide our Drupal site and any themes and modules into the container.
      
        volumes:   db_data: 

This tells Docker Compose to create the volume we specified earlier in the config.
Create App Directory Here’s the hardest step, create an app directory at the root of our project.

        mkdir app 

Wheww.
Save Our Progress With Git
At this time, we can create our Git repo and save our progress. Doing this now isn’t necessary but it allows us to visualize how the filesystem changes during the next several steps.

        git init . 
        git add . 
        git commit -m "Initial commit." 
        
Building The Drupal 8 Website
With the initial scaffolding out of the way, we can go ahead and get our actual site built. We start with bringing up our containers with Docker Compose.

      docker-compose up -d --build 

--build tells Docker Compose to build our Dockerfile fresh when we run this command. We need to login to our main container to run Composer, but we need the container name to do so. We can find it by running docker-compose ps. The container connected to port 80 is the correct one. Here’s an example of the output:

    $ docker-compose ps   
    
    Name            Command                         State           Ports          
    ---------------------------------------------------------------------------------------  
    rsas_db_1       docker-entrypoint.sh mysqld      Up             3306/tcp
    rsas_drupal_1   docker-php-entrypoint apac ...   Up      0.0.0.0:8080->80/tcp 

The correct container name here is rsas_drupal_1. We log into it using the docker exec command.

    docker exec -it rsas_drupal_1 bash 
    
This will drop you into the container at the /app directory. Now we can use composer to install Drupal.

    /app #  composer create-project drupal-composer/drupal-project:8.x-dev /app --stability dev --no-interaction 
    /app #  mkdir -p /app/config/sync 
    /app #  chown -R www-data:www-data /app/web 
    
Now visit http://localhost:8080 in your browser and go through the Drupal installer.
Once everything is installed and your site is good to go, we’re going to use Drush to export your Drupal configuration. Then, exit the container.

    /app #  drush config-export 
    /app #  exit 

We’re going to edit the .gitignore file that we have as well to remove lines 11 and 15. These are the lines ignoring the settings.php file and the */files/* directories. Some people would say don’t do this, and in certain circumstances they’re right. To keep things simple, we’re going to version the files directory here and we’re versioning settings.php. Don’t version this file if your repo is going to be public as everyone will have your database credentials.
Running git status will show you everything that Composer has installed. We can commit everything with Git and we’re done. We now have the saved initial state of our Drupal website.
Where To Go From Here?
We now have a basic Drupal 8 website tracked with Git and Composer. We can even use Docker Compose to get that site up and running at any location (if we also export & import the database as well).
Our next blog post in this series will cover how can we get our Drupal website in a Continuous Integration workflow on CircleCI to do some basic testing before we publish our site to production.
In the meantime, here are other things you can do with our setup.

Updating Drupal 8 Core

I suggest creating a new Git branch before updating Core (or most changes really). Drupal 8 Core can be updated as follows:

        git checkout -b update-drupal 

        docker exec -it rsas_drupal_1 bash 

        /app #  composer update drupal/core --with-dependencies 
        /app #  drush updb  # updates the DB with any schema changes 

        /app #  git diff    # to check for changes 

        /app #  git status  # also to help review changes 
    
Test our the site in a browser (and in our next article, with CI) to make sure everything works as expected.

        /app #  exit 

        git add . 
        git commit -m "Updated Drupal Core." 
        git push 

Installing a “Contrib” Module or Theme
“Contrib” modules are public modules that someone published or “contributed” to the official Drupal registry on Drupal.org. These modules can also be installed and tracked with Git and Composer. Here we’ll install the Pathauto Drupal module.

        git checkout -b install-pathauto 

        docker exec -it rsas_drupal_1 bash
        
        /app #  composer require drupal/pathauto 
        /app #  drush en pathauto -y  # enables the module with Drush rather than visiting the Drupal Admin page 
      
Test our the site in a browser (and in our next article, with CI) to make sure everything works as expected. If you do any configuration for the new module, don’t forget to export your config with Drush.

        /app #  drush config-export  # if you made config changes 
        /app #  exit 


        git add . 
        git commit -m "Installed Pathauto." 
        git push 
    
Updating Drupal “Contrib” Modules & Themes
Contrib modules can be updated similarly to Core. At anytime, you can run composer outdatedto see every piece of your Drupal site that has an update available.

        git checkout -b update-pathauto 
        
        docker exec -it rsas_drupal_1 bash 
        
        /app #  composer update drupal/pathauto --with-dependencies 
        /app #  drush updb  # updates the DB with any schema changes 

Test our the site in a browser (and in our next article, with CI) to make sure everything works as expected.

        /app #  exit 
        
        git add . 
        git commit -m "Updated Pathauto." 
        git push 

Maintaining Custom Drupal Modules & Themes
There’s two ways to do this. If you have a module that is specific to this site, then you can keep all of the code within the same Git repo and maintain it here. Think about this though because many times modules can serve multiple “sites within the same company, or end up getting open-sourced which is always an awesome thing to do. Themes are more likely to be site specific but the same warning applies.
For modules and themes that are not site specific, instead of maintaining the code within this repo, you’d likely keep them in their own repository and add them as a submodule to your Drupal site.
If you do that, whenever you checkout your Drupal 8 site’s repository, you’ll want to run the following commands to make sure your custom modules get checked out as well:

        git submodule sync 
        git submodule update --init 
        
To update them, you’d cd into their directories and then run the normal git pull or git checkout <tagname>. Then, back up to the root of this repository and commit your changes.

--- End of Blog 1 ---



