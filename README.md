# Linux Server Configuration

## Server Info

SSH: `ssh -i ~/.ssh/udacity_key.rsa root@52.36.113.208`

Catalog App: [http://52.36.113.208](http://52.36.113.208)

## Configuration Summary

1. Connect to VM:

		ssh -i ~/.ssh/udacity_key.rsa root@52.36.113.208

2. Create a new user named `grader`:

		adduser grader

3. Give `grader` the permission to sudo:

		cd /etc/sudoers.d/
		touch grader
		vi grader

	Add the followings:

		grader ALL=(ALL) NOPASSWD:ALL
		
	Generate a key pair on my local machine:
		
		ssh-keygen
	
	Install a public key on the server(log in as `grader`):
		
		mkdir .ssh
		touch .ssh/authorized_keys
	
	Paste contents of the public key into `authorized_keys`.
	
	Change access permissions:
	
		chmod 700 .ssh
		chmod 644 .ssh/authorized_keys
	
	Now log in again as `grader`:
		
		ssh -i ~/.ssh/udacity grader@52.36.113.208 -p 2200


4. Update all currently installed packages

		sudo apt-get update
		sudo apt-get upgrade

5. Change the SSH port from 22 to 2200, and disable root login:

        vi /etc/ssh/sshd_config
        
	Modify the following:
		
		Port 2200
		PermitRootLogin	no
	
	Restart ssh service:
	
		sudo service ssh restart

6. Configure UFW to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123):
		
		sudo ufw allow 2200/tcp
		sudo ufw allow 80/tcp
		sudo ufw allow 123/tcp
		sudo ufw enable
		
7. Configure the local timezone to UTC:

		date
		
	It turned out that it was UTC already.

8. Install Apache, mod_wsgi, postgresql and git:

		sudo apt-get install apache2 apache2-dev python-dev libapache2-mod-wsgi postgresql git

9. Retrieve `item-catalog` project:

		git clone https://github.com/zhihaodev/udacity-projects.git
		sudo cp -R udacity-projects/item-catalog /var/www
	
	Rename for convience:
		
		cd /var/www
		sudo mv item-catalog catalog
		
10. Create a user `catalog` and grant permissions to the database:

		sudo adduser catalog
		sudo su - postgres
		psql
		CREATE USER catalog;
		CREATE DATABASE catalog;
		GRANT ALL PRIVILEGES ON DATABASE catalog to catalog;
		\q
		
	Remote connections are not allowed to the database by default when installing PostgreSQL from the Ubuntu repositories.


11. Modify `item-catalog` to use PostgreSQL instead of SQLite3:
		
		vi catalog/config.py
		
	Add one line:
		
		# SQLALCHEMY_DATABASE_URI = 'sqlite:///' + basedir + '/catalog.db'
		SQLALCHEMY_DATABASE_URI = 'postgresql:///catalog'	

12. Set up a virtual environment for the Flask application:
	
	Install:

		sudo apt-get install python-pip
		sudo pip install virtualenv

	Activate `virtualenv`:
	
		virtualenv venv
		source venv/bin/activate
	
	Install all requirements:
	
		pip install -r catalog/requirements.txt

13. Configure Apache appropriately:

		sudo vi /etc/apache2/sites-available/catalog.conf
		
	Add the followings:
	
		<VirtualHost *:80>
	    ServerName catalog.com

    	WSGIDaemonProcess catalog user=catalog group=catalog threads=5
	    WSGIScriptAlias / /var/www/catalog/catalog.wsgi

    	<Directory /var/www/catalog>
        	WSGIProcessGroup catalog
	        WSGIApplicationGroup %{GLOBAL}
    	    Order deny,allow
        	Allow from all
	    </Directory>

    	ErrorLog ${APACHE_LOG_DIR}/error.log
	    LogLevel warn
    	CustomLog ${APACHE_LOG_DIR}/access.log combined
	</VirtualHost>
	
	Create the .wsgi file:
		
		sudo vi catalog/catalog.wsgi
		
	Add the followings:
		
		activate_this = '/home/grader/venv/bin/activate_this.py'
		execfile(activate_this, dict(__file__=activate_this))
		import sys
		import os
		sys.path.insert(0, '/var/www/catalog')
		os.chdir("/var/www/catalog")
		from manage import app as application
	
	Disable the default virtualhost and enable ours:
		
		sudo a2dissite default
		sudo a2ensite catalog
		sudo service apache2 restart

14. Go to Google Developers Console and add [http://52.36.113.208](http://52.36.113.208) and [https://52.36.113.208](https://52.36.113.208) to Authorized JavaScript origins.

15. Done! Checkout [http://52.36.113.208](http://52.36.113.208).
		

## References

1. [http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/#configuring-apache](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/#configuring-apache)
2. [http://flask.pocoo.org/snippets/99/](http://flask.pocoo.org/snippets/99/)
3. [https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
4. [https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
5. [http://killtheyak.com/use-postgresql-with-django-flask/](http://killtheyak.com/use-postgresql-with-django-flask/)
5. [http://www.ruanyifeng.com/blog/2013/12/getting_started_with_postgresql.html](http://www.ruanyifeng.com/blog/2013/12/getting_started_with_postgresql.html)
