# Linux Server Configuration:


Took a baseline installation of a Linux server and prepared it to host my Item Catalog web applications. Secured the server from a number of attack vectors, installed and configured a database server, and deployed the Item Catalog web applications onto it.

You can visit http://18.221.244.17/ for the website deployed.

The EC2 URL is http://ec2-18-221-244-17.us-east-2.compute.amazonaws.com/ and the local IP address is http://18.221.244.17

## Login as root to server

1. Login as root to server after creating the server instance on Amazon Lightsail

## Create a new user named grader

1. `sudo adduser grader`
2. `sudo vi /etc/sudoers`
3. `sudo touch /etc/sudoers.d/grader`
4. `sudo nano /etc/sudoers.d/grader`, type in `grader ALL=(ALL:ALL) ALL`, save and quit

## Instructions for SSH access to the instance

1. Using ssh-keygen on your local machine, create a key for the user grader. 
Follow the prompts to provide a name for this key and use the default key location (~/.ssh). 
This process will create two keys on your local machine, the file with extension .pub is the public key to be transferred to the server.`

## Set ssh login using keys

1. Generate keys on local machine using`ssh-keygen` ; then save the private key on local machine
2. Deploy public key on developement enviroment

	On you virtual machine:
	```
	$ su - grader
	$ mkdir .ssh
	$ touch .ssh/authorized_keys
	$ nano .ssh/authorized_keys
	```
	Copy the public key (_one with the extension .pub_) generated on your local machine to this file and save.
  Then type the following commands:
	```
	$ chmod 700 .ssh
	$ chmod 644 .ssh/authorized_keys
	```
	
3. reload SSH using `service ssh restart`
4. now you can use ssh to login with the new user you created

	`ssh grader@18.221.244.17 -i ~/.ssh/[privateKeyFilename]`

## Update all currently installed packages

	sudo apt-get update
	sudo apt-get upgrade

## Change the SSH port from 22 to 2200

1. Use `sudo nano /etc/ssh/sshd_config` and then change Port 22 to Port 2200 , save & quit.
2. Reload SSH using `sudo service ssh restart`
3. Now try 'exit' to logout and try relogging in using `ssh grader@18.221.244.17 -p 2200 -i ~/.ssh/[privateKeyFilename]`

__Note:__ Remember to add and save port 2200 with _Application __as__ Custom and Protocol __as__ TCP_ in the Networking section of your instance on Amazon Lightsail. 

## Configure the Uncomplicated Firewall (UFW)

Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

1. sudo ufw allow ssh
2. sudo ufw allow www
3. sudo ufw allow ntp
4. sudo ufw allow 2200/tcp
5. sudo ufw allow 80/tcp
6. sudo ufw allow 123/udp
7. sudo ufw deny 22 to disable port 22 
8. sudo ufw enable 
9. sudo ufw status to check firewall status and settings
 
## Configure the local timezone to UTC

1. Configure the time zone `sudo dpkg-reconfigure tzdata` and select other and choose UTC.
2. It is now set to UTC.

## Install and configure Apache to serve a Python mod_wsgi application

1. Install Apache `sudo apt-get install apache2`
2. Install mod_wsgi `sudo apt-get install python-setuptools libapache2-mod-wsgi`
3. Restart Apache `sudo service apache2 restart`

## Install and configure PostgreSQL

1. Install PostgreSQL `sudo apt-get install postgresql`
2. Check if no remote connections are allowed `sudo vi /etc/postgresql/9.3/main/pg_hba.conf`
3. Login as user "postgres" `sudo su - postgres`
4. Get into postgreSQL shell `psql`
5. Create a new database named catalog  and create a new user named catalog in postgreSQL shell
	
	```
	postgres=# CREATE DATABASE catalog;
	postgres=# CREATE USER catalog;
	```
5. Set a password for user catalog
	
	```
	postgres=# ALTER ROLE catalog WITH PASSWORD 'password';
	```
6. Give user "catalog" permission to "catalog" application database
	
	```
	postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
	```
7. Quit postgreSQL `postgres=# \q`
8. Exit from user "postgres" 
	
	```
	exit
	```
 
## Install git, clone and setup your Catalog App project.
1. Install Git using `sudo apt-get install git`
2. Use `cd /var/www` to move to the /var/www directory 
3. Create the application directory `sudo mkdir catalog`
4. Move inside this directory using `cd catalog`
5. Clone the Item-Catalog App to the virtual machine `sudo git clone https://github.com/karthic-psa/Item-Catalog.git catalog`
6. Now the catalog app is the directory /var/www/catalog/catalog
7. The reference document to create and run Flask apps <https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps>
8. Rename `python.py` to `__init__.py` using `sudo mv python.py __init__.py`, if `__init__.py` not present.
9. Edit `database_setup.py` and `users.py` to change `engine = create_engine('sqlite:///restaurantmenuwithusers.db.db')` to `engine = create_engine('postgresql://catalog:password@localhost/catalog')`, if not already done.
10. Install pip `sudo apt-get install python-pip`
11. Use pip to install dependencies -
	* `sudo pip install sqlalchemy flask-sqlalchemy psycopg2 bleach requests`
	* `sudo pip install flask packaging oauth2client redis passlib flask-httpauth`
13. Install psycopg2 `sudo apt-get -qqy install postgresql python-psycopg2`
14. Create database schema `sudo python database_setup.py`
15. Populate the database using `sudo python users.py`
16. Run the app by using `sudo python __init__.py`


## Configure and Enable a New Virtual Host
1. Create catalog.conf to edit: `sudo nano /etc/apache2/sites-available/catalog.conf`
2. Add the following lines of code to the file to configure the virtual host. 
	
```
<VirtualHost *:80>
    ServerName 18.221.244.17
    ServerAlias ec2-18-221-244-17.us-east-2.compute.amazonaws.com
    ServerAdmin karthic.psa@gmail.com
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
```
3. Enable the virtual host with the following command: `sudo a2ensite catalog` reload apache2 using `sudo service apache2 reload`

## Create the .wsgi File
1. Create the .wsgi File under /var/www/catalog: 
	
	```
	cd /var/www/catalog
	sudo nano catalog.wsgi 
	```
2. Add the following lines of code to the catalog.wsgi file:
	
```
#!/usr/bin/python
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

## Restart Apache
1. Restart Apache `sudo service apache2 restart`
2. Run the app by using `sudo python __init__.py`

**The Google Sign In button is not working on Google Chrome browser, so please use any other browser.**  

## References:
1. Udacity's Full-Stack Web Developer course and forums
2. https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
3. https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
