# LinuxServerConfiguration-Udacity

## Project Overview
In this project, we take a baseline installation of a Linux server and prepare it to host our web applications. We will secure our server 
from a number of attack vectors, install and configure a database server, and deploy Item catalog web applications onto it.

* Public IP Address: 54.149.159.48
* Accessible SSH Port: 2200
* Website deployed at URL: http://54.149.159.48

## Steps followed:
### Accessing Linux instance
1. Download private key from AWS Lightsail instance from account page and copy it to ~/.ssh folder where ~ is environment's home directory
and rename it as udacity_key.rsa
2. On local terminal, type $ chmod 600 ~/.ssh/udacity_key.rsa
3. On local terminal, type $ ssh -i ~/.ssh/udacity_key.rsa ubuntu@54.149.159.48

### Create user grader
1. On the connected server, create a user named grader: $ sudo adduser grader
2. $ sudo nano /etc/sudoers.d/grader
Copy grader ALL=(ALL:ALL) ALL in above file, save and exit

### Key-based authentication
1. In local machine, generate key pairs by
ssh-keygen
Enter the passphrase that you want to give, for no passphrase, just press enter.
Save the private key in ~/.ssh on local machine.
2. Login as grader on the virtual machine:
  $ su - grader
3. Make a directory .ssh
$ mkdir .ssh
4. $ touch .ssh/authorized_keys
   $ nano .ssh/authorized_keys
5. On the local machine, get the public key by executing:
   $ cat .ssh/id_rsa.pub
   Copy above public key and paste it in authorized_keys file above and save the file.
6. Set up file permissions on above files:
   $ chmod 700 .ssh
   $ chmod 644 .ssh/authorized_keys
7. Restart the service to reload ssh:
   $ sudo service ssh restart
8. Now you can use ssh to login with user grader on the local machine:
  $ ssh grader@54.149.159.48 -i ~/.ssh/id_rsa

### Update all currently installed packages
$ sudo apt-get update
$ sudo apt-get upgrade

### Configure UFW and change SSH port from 22 to 2200
1. In AWS Lightsail, under Networking -> Firewall, add Custom -> TCP -> 2200 and save.
2. In virtual terminal, type:
$ sudo nano /etc/ssh/sshd_config
Do not remove Port 22 now. Just add Port 2200 below Port 22 in above file, save and exit.
3. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH(Port 2200), HTTP(Port 80), and NTP (Port 123). This will be done on virtual machine.
$ sudo ufw status
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow ssh
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/tcp
$ sudo ufw allow www
- Block anything from Port22
$ sudo ufw deny 22
$ sudo ufw enable
$ sudo ufw status

UFW is enabled now.
4. Reboot server from AWS website - reboot button
5. Now we should be able to login on Port 2200 from a local terminal using:
$ ssh grader@54.149.159.48 -p 2200 -i ~/.ssh/id_rsa
$ ssh <your_username>@<your_public_ip> -p 2200 -i ~/.ssh/<your_private_key>
6. If above works, remove Port 22 from sshd_config file and from AWS firewall settings.

### Configure local timezone to UTC
1. Configure the timezone:
$ sudo dpkg-reconfigure tzdata
2. It is already set to UTC

### Install and configure Apache to serve a Python mod_wsgi application
1. Install Apache 
$ sudo apt-get install apache2
2. Install mod_wsgi 
$ sudo apt-get install python-setuptools libapache2-mod-wsgi
3. Restart Apache 
$ sudo service apache2 restart

### Install and configure PostgreSQL
1. Install PostgreSQL 
$ sudo apt-get install postgresql
2. Login as user "postgres" 
$ sudo su - postgres
3. Get into postgreSQL shell 
   psql
4. Create a new database named catalog and create a new user named catalog in postgreSQL shell
   postgres=# CREATE DATABASE catalog;
   postgres=# CREATE USER catalog;
5. Set a password for user catalog
   postgres=# ALTER ROLE catalog WITH PASSWORD 'password';
6. Give user "catalog" permission to "catalog" application database
   postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
7. \c catalog
  REVOKE ALL ON SCHEMA public FROM public;
  GRANT ALL ON SCHEMA public TO catalog;
8. Quit postgreSQL postgres=# \q
   Exit from user "postgres"

### Install git, clone and setup your Item Catalog project
1. Install Git using 
$ sudo apt-get install git
2. Use cd /var/www to move to the /var/www directory
3. Create application directory 
$ sudo mkdir catalog
4. Change owner of the newly created catalog folder 
$ sudo chown -R grader:grader catalog
$ cd catalog
5. Clone your project from github 
$ git clone https://github.com/anupandey20/Udacity_FSND_Item_Catalog_Project.git catalog
6. Rename application.py to init.py 
$ mv application.py __init__.py
7. Change create_engine line in your __init__.py, catalogCategories.py and database_setup.py to: 
engine = create_engine('postgresql://catalog:password@localhost/catalog')
8. Install pip 
$ sudo apt-get install python-pip
9. Install psycopg2 
$ sudo apt-get -qqy install postgresql python-psycopg2
10. Create database schema 
$ sudo python database_setup.py
11. Add in catalog by executing: 
$ sudo python catalogCategories.py

### Create the .wsgi file
1. Create a catalog.wsgi file:
$ sudo nano catalog.wsgi
2. Add below code inside above file:
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'item_secret_key'
3. Restart apache
$ sudo service apache2 restart

### Configure and Enable a New Virtual Host
1. Run this $ sudo nano /etc/apache2/sites-available/catalog.conf
2. Copy below content and save the file:
<VirtualHost *:80>
    ServerName 54.149.159.48
    ServerAdmin admin@54.149.159.48
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
3. Enable the virtual host 
$ sudo a2ensite catalog
4. Restart apache
$ sudo service apache2 restart
5. Visit site at http://54.149.159.48

