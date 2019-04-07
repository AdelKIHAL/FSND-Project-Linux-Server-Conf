# FSND-Linux Server Configuration Project
## By Adel KIHAL


# Project Goal
> Host a flask application connected to postgresql database
> Create an admin user and ensure a secure remote access to the server

# Connection info

> DDNS: kadev.hopto.org
>
> SSH Port: 2200
>
>	Website URL: http://kadev.hopto.org


# Configuration:

Due the lack of a credit card Oo, I used what I had available, so:

#### 1. Set Up a VM
- Install virtual box
- Create a new VM and go to Settings->Network and attach it to:Bridged Adapter
- In the Settings go to System and change the Chipset to ICH9 to increase network speed. source( [networking - Slow download on VirtualBox Linux guest - Super User](https://superuser.com/questions/1235168/slow-download-on-virtualbox-linux-guest) )
- Install Ubuntu server
- Update your system
```bash
sudo apt update && sudo upgrade -y
```
- Configure your timezone
```bash
sudo dpkg-reconfigure tzdata
```
- Install a dynamic dns client. source( https://www.noip.com/support/knowledgebase/installing-the-linux-dynamic-update-client-on-ubuntu/ )

#### 2. New user
- Create a grader user
```bash
sudo adduser grader
```
- Add it to sudoers
```bash
sudo echo "grader ALL=(ALL:ALL) ALL" > /etc/sudoers.d/grader
```
### 3. Create SSH keys to login the new user
-  Generate a key pair (on your local machine)
```bash
ssh-keygen -f ~/.ssh/grader_on_kadev
```
- Copy the public key to the remote server
```bash
ssh-copy-id -i ~/.ssh/grader_on_kadev.pub grader@kadev.hopto.org
```
- You can now login by using
```bash
ssh grader@kadev.hopto.org -i ~/.ssh/grader_on_kadev
```
- Disable root,using password and change sshd port to 2200
```bash
sudo vim /etc/ssh/sshd_config
```
> change PermitRootLogin to PermitRootLogin no
>
> change PasswordAuthentication to PasswordAuthentication no
>
> change Port to Port 2200

- restart ssh service
```bash
sudo systemctl restart ssh
```
- Now use this command to login again
```bash
ssh grader@kadev.hopto.org -i ~/.ssh/grader_on_kadev -p 2200
```
#### 4. UFW Configuration
- Check ufw status, then disable upcomming traffic and enable outgoing trafic, after that open needed port and enable ufw.

```bash
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp2200.
sudo ufw allow www
sudo ufw allow 123/udp
sudo ufw enable
```
    To                         Action      From
    --                         ------      ----
    2200/tcp                   ALLOW       Anywhere
    80/tcp                     ALLOW       Anywhere
    123/udp                    ALLOW       Anywhere
    2200/tcp (v6)              ALLOW       Anywhere (v6)
    80/tcp (v6)                ALLOW       Anywhere (v6)
    123/udp (v6)               ALLOW       Anywhere (v6)

- Configure your router to forward these ports to your local ip

#### 5. Deploying the application

##### A. The Setup

- Install Apache2, mod_wsgi for python3
```bash
sudo apt install apache2 libapache2-mod-wsgi-py3
```
- Install postgresql
```bash
sudo apt install postgresql libpq-dev postgresql-client postgresql-client-common
```
##### B. The configuration
 - Add a new user to postgresql and allow it to admin a new db
```sql
sudo su - postgres
psql
psql=# create database catalog_db;
psql=# create user catalog_db_admin with encrypted password 'pass';
``` 
- Download the project
```bash
cd /var/www
sudo mkdir catalog && cd catalog
sudo git clone https://github.com/AdelKIHAL/FSND-Project-Sport-Catalog.git catalog
sudo mv catalog.py __init__.py
```
- Edit the files to change connection from sqlite3 to postresql
> databse_setup.py
>
> populate_db.py
>
> __init__.py


```python
engine = create_engine('postgresql+psycopg2://catalog_db_admin:SECRET@localhost:5432/catalog_db')
```

- add catalog-app.wsgi file to the parent directory
```bash
sudo vim ../catalog-app.wsgi
```
```python
#!/usr/bin/python

import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
```
- Edit you apache sites available to add the catalog app
```bash
sudo vim /etc/apache2/sites-available/sports-catalog.conf
```
```bash
<VirtualHost *:80>
        ServerName kadev.hopto.org

### Catalog project config
    WSGIDaemonProcess catalog python-path=/var/www/catalog/catalog:/var/www/catalog/catalog/venv/lib/python3.6/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog-app.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>

    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>

    Alias /media /var/www/catalog/catalog/media
    <Directory /var/www/catalog/catalog/media/>
        Order allow,deny
        Allow from all
    </Directory>
        ServerAdmin kihal.adel@gmail.com
        DocumentRoot /var/www/html
LogLevel warn
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- Disable default web site and activate the new one
```bash
sudo a2dissite 000-default.conf
sudo a2ensite sports-catalog.conf
```
- Install venv and activate it to install project depenencies
```bash
sudo apt install python3-pip
sudo pip install virtualenv 
cd /var/www/catalog/catalog
sudo virtualenv venv
source venv/bin/activate
pip install -r requirements.txt
```
- Activate apache new config
```bash
sudo apachectl restart
```
source ( https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps )

	
#### Visit http://kadev.hopto.org

