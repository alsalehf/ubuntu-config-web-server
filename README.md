# Linux Server Configuration
This is a project to set up a Linux server (ubuntu 16.04). This server is run by Lightsail, an Amazon Web Service. It is configured to host the Item Catalog web application, which a project part of the Udacity Full Stack Developer Nanodegree.

The item catalog is a web application that displays item catalog for different types of sports allowing the user to login via Google and add, manage items they add. It interacts with a postgresql database using SQLAlchemy. Python Flask framework is used to manage the web pages. OAuth 2.0 framework is used to authenticate users using Google.</br>
Public IP: 52.58.155.128</br>
SSH port: 2200</br>
Website URL: http://52.58.155.128.xip.io</br>
JSON Endpoint: http://52.58.155.128.xip.io/catalog.json</br>


## Server Configuration Steps
### 1 -  SSH from local machine to the lightsail virtual server instance
* Download the AWS provided default private key to the local machine</br>
* Change the private key file permission not to be accessible by others `chmod 600 LightsailDefaultPrivateKey-eu-central-1.pem`</br>
* Logged in, ssh successful ` ssh ubuntu@52.58.155.128 -p 22 -i ~/Downloads/LightsailDefaultPrivateKey-eu-central-1.pem`

### 2 -  Install updates
* Updates the list of available packages and versions for upgrade `sudo apt-get update`</br>
* Installs newer versions of the packages `sudo apt-get upgrade`</br>

### 3 - Create grader user
* Create new user grader `sudo adduser grader`</br>
* Edit the file grader to add the sudo access `sudo nano /etc/sudoers.d/grader`</br>
   . grader ALL=(ALL) NOPASSWD:ALL</br>
   . Save changes

### 4 - Create an SSH key pair for grader using the ssh-keygen tool
* On the local machine terminal, generate key pair using `ssh-keygen`
* Named it udacityGraderKey.pub
* Save the private key in `~/.ssh` on local machine
* On the server instance:
    ```
    su - grader
    mkdir .ssh
    touch .ssh/authorized_keys
    nano .ssh/authorized_keys

    Copy the public key content from local machine to this file and save, change the access level
    chmod 700 .ssh
    chmod 644 .ssh/authorized_keys
    ```

### 5 - Change the SSH port from 22 to 2200
* Configure the Lightsail firewall in AWS console to allow incoming connections into ports HTTP (port 80), and NTP (port 123)
* Configure the Lightsail firewall in AWS console to allow port 2200 ssh connections
* Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
    ```
    sudo ufw allow 2200/tcp
    sudo ufw allow www
    sudo ufw allow 123/udp
    sudo ufw deny 22
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw enable
    sudo ufw status
    ```
* Make changes in the config file in the server from port 22 to 2200 ```sudo nano /etc/ssh/sshd_config```
* Restart SSH `sudo service ssh restart`
* Prevent remote root login: ```sudo nano /etc/ssh/sshd_config``` `PermitRootLogin no`
* SSH into grader using the key on 2200 port `ssh grader@52.58.155.128 -p 2200 -i ~/.ssh/udacityGraderKey`

### 6 - Prepare to deploy the project
  SSH to the server</br>
#### Configure the local timezone to UTC
* Check current timezone setting `sudo cat /etc/timezone`
* Configure the local timezone to UTC `sudo dpkg-reconfigure tzdata`

#### Install and configure Apache to serve a Python mod_wsgi application
* Install mod_wsgi package(although using python3, this serves well without causing errors) `sudo apt-get install apache2 apache2-utils libexpat1 ssl-cert python`
* Install wsgi `sudo apt-get install libapache2-mod-wsgi`
* Once done http://52.58.155.128 serves the default apache page

#### Install and configure PostgreSQL
  `sudo apt-get install postgresql`
* Do not allow remote connections (it's already blocked)
   ```
    sudo ls /etc/postgresql/9.5/main
    sudo cat /etc/postgresql/9.5/main/pg_hba.conf
    sudo service apache2 restart
   ```

* Create a new database user named catalog that has limited permissions to catalog application database
    ```
    sudo su - postgres
    CREATE USER catalog WITH PASSWORD 'catalogUser';
    \du
    CREATE DATABASE catalog;
    \l
    ```

### 7 - Deploy the Item Catalog app from git and create the required set up on server
* Create the FlaskApp, cloning the git repository
    ```
    cd /var/www
    sudo mkdir CatalogApp
    cd CatalogApp
    sudo git clone https://github.com/alsalehf/item-catalog-project.git
    ```
* Make changes for the app in Google API Console to accept the lightsail public IP, URL and redirect URIs. I used the xip.io as my domain with my IP address: 52.58.155.128.xip.io.
* Edit the python files to reflect the correct postgresql database connection - username, password, DBname
    `sudo nano item-catalog.py`
    `sudo nano database-setup.py`
    `sudo nano seeder.py`
    change engine = create_engine('sqlite:///itemcatalog.db') to engine = create_engine('postgresql://catalog:catalogUser@localhost/catalog')

* Rename item-catalog.py to itemCatalog.py using `sudo mv item-catalog.py itemCatalog.py`

#### Install Flask
Setting up a virtual environment will keep the application and its dependencies isolated from the main system.
    sudo apt-get install python-pip
    sudo pip install virtualenv
    sudo virtualenv venv
    source venv/bin/activate
    sudo pip install Flask
    pip install httplib2
    sudo apt-get install python-oauth2client
    sudo apt-get install python-requests
    sudo apt-get install  python-sqlalchemy
    sudo apt-get python-psycopg2
    sudo python itemCatalog.py

#### Enable the FlaskApp

    sudo nano /etc/apache2/sites-available/FlaskApp.conf

    <VirtualHost *:80>
        ServerName 52.58.155.128
        WSGIScriptAlias / /var/www/CatalogApp/flaskapp.wsgi
        <Directory /var/www/CatalogApp/item-catalog-project/>
            Order allow,deny
            Allow from all
        </Directory>
        Alias /static /var/www/CatalogApp/item-catalog-project/static
        <Directory /var/www/CatalogApp/item-catalog-project/static/>
            Order allow,deny
            Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        </VirtualHost>

     sudo a2ensite FlaskApp

#### Create the .wsgi File

`cd /var/www/CatalogApp`
`sudo nano flaskapp.wsgi`

Add the following lines of code to the flaskapp.wsgi file

    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/CatalogApp/item-catalog-project")

    from itemCatalog import app as application
    application.secret_key = 'super_secret_key'

#### Restart Apache
`sudo service apache2 restart`
Check error log for errors if any, features not working: `sudo cat /var/log/apache2/error.log`

#### Add tables and data
Run database setup file: `sudo python /var/www/CatalogApp/item-catalog-project/database-setup.py`</br>
Feed the database: `sudo python /var/www/CatalogApp/item-catalog-project/seeder.py` </br>

* Resources
Udacity forums</br>
https://help.ubuntu.com/</br>
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04</br>
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps</br>
https://devops.profitbricks.com/tutorials/install-and-configure-mod_wsgi-on-ubuntu-1604-1/</br>
http://xip.io/</br>
