# Ubuntu NGINX Gunicorn Supervisor Django React Server setup #

## Deployment to a VPS

Log in to the server using the terminal.
```shell
ssh root@xx.xx.xxx.xx
```
Set the new password, before configuring the server (if new box).
Run commands below to update server (ubuntu).
```shell
sudo apt update
sudo apt -y upgrade 
```
## Install Python 3.x
Ubuntu 20.x comes with Python 3 pre-installed.
```shell
sudo apt update
python -V
sudo apt install -y python3-pip
```
## Install PostgreSQL
```shell
sudo apt -y install postgresql postgresql-contrib
```
## Install  NGINX
```shell
sudo apt -y install nginx
```
## Install Supervisor
```shell
sudo apt -y install supervisor
sudo systemctl enable supervisor
sudo systemctl start supervisor
```
## Install Virtualenv
```shell
sudo apt install python3-venv
```
## Create New Application User

Choose the name of the application. Enter a password and optionally add extra info at the prompt.	
```shell
adduser  app-name
```
Add the user to the sudoers list.
```shell
gpasswd -a app-name sudo
```

## PostgreSQL Database Setup

Switch to the postgres user.
```shell
sudo su – postgres
```
Create a database user.
```shell
createuser u_ app-name
```
Create a new database and set the user as the owner.
```shell
createdb app-name --owner u_app-name
```
Define a strong password for the user.
```shell
psql -c "ALTER USER u_app-name WITH PASSWORD 'OcexPRVAqCvE7RMgBPzxcAZoYWsJb'"
```
Now exit the postgres user.
```shell
exit
```

## Django Project Setup

Switch to the application user.
```shell
sudo su - app-name
```
Check where we are in the system.		
```shell
pwd
```
Output:
```shell
$ home/app-name
```

Clone the repository with our code.
```shell
git clone https://github.com/username/project.git
```
Start a virtual environment.	
```shell
virtualenv venv
```
Initialize the virtualenv.	
```shell
source venv/bin/activate
```
Install the requirements.		
```shell
pip install -r requirements.txt
```
Additional libraries to add, Gunicorn and the PostgreSQL drivers.
```shell
pip install gunicorn
pip install psycopg2
```
Inside the /home/app-name/app-name/ project directory, create the .env file to store the database credentials, secret key and other sensitive data.

    SECRET_KEY=syf2p=n&@0xsp87m$@!kp=df_wrqr_cjv4igscy
    ALLOWED_HOSTS=.yourdomain.com
    DATABASE_URL=postgres://u_app-name:OcexPRVAqCvE7RMgBPzxcAZoYWsJb@localhost:5432/dbname

Example DB connection string.

    postgres://db_user:db_password@db_host:db_port/db_name

Migrate the database, collect the static files and create a superuser.
```shell
cd app-name
./manage.py check --deploy
./manage.py migrate
./manage.py collectstatic
./manage.py createsuperuser	
```

## Configuring Gunicorn

Gunicorn is responsible for executing the Django code behind a proxy server.
Create a new file named gunicorn_start inside /home/app-name/app-name/	

	#!/bin/bash 
	NAME="app-name" 
	DIR=/home/app-name/app-name/project 
	USER=app-name 
	GROUP=app-name 
    WORKERS=3 
	BIND=unix:/home/app-name/app-name/run/gunicorn.sock 
	DJANGO_SETTINGS_MODULE=project.settings 
	DJANGO_WSGI_MODULE=project.wsgi 
	LOG_LEVEL=error 
	
	cd $DIR 
	source ./venv/bin/activate 

	export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE 
	export PYTHONPATH=$DIR:$PYTHONPATH 

	exec ./venv/bin/gunicorn ${DJANGO_WSGI_MODULE}:application \ 
	    --name $NAME \ 
	    --workers $WORKERS \ 	
	    --user=$USER \	
	    --group=$GROUP \ 
	    --bind=$BIND \ 
	    --log-level=$LOG_LEVEL \ 
	    --log-file=-

This script will start the application server. Provide information such as where the Django project is, and which application user to be used to run the server.

Make the file executable.
```shell
chmod u+x gunicorn_start
```
Create two directories, one for the socket file and one to store the logs.
```shell
mkdir run logs
```

The directory structure inside /home/app-name/ should look like this.
	
    app-name/
	gunicorn_start
	logs/
	run/
	staticfiles/
	venv/

## Configuring Supervisor

Create an empty log file inside the /home/app-name/app-name/logs/ directory.

```shell
	
touch gunicorn.log
```

Create a new supervisor file.
```shell
sudo nano /etc/supervisor/conf.d/app-name.conf
```

Add the below details.
```shell
[program:app-name] 
command=/home/app-name/app-name/gunicorn_start 
user=app-name 
autostart=true 
autorestart=true 
redirect_stderr=true 
stdout_logfile=/home/app-name/app-name/logs/gunicorn.log
```
Save the file and run the commands below.
```shell
sudo supervisorctl reread
sudo supervisorctl update
```

Check the status.
```shell
sudo supervisorctl status app-name
```
Output:
```shell
$ app-name		RUNNING pid 333, uptime 0:00:07
```

## Configuring NGINX

Set up the NGINX server to serve the static files and to pass the requests to Gunicorn.
Add a new configuration file named app-name inside /etc/nginx/sites-available/.

	upstream app_server { 
		server unix:/home/app-name/app-name/run/gunicorn.sock fail_timeout=0; 
	} 
	server { 
		listen 80; 
		server_name www.yourdomain.com; #  or IP address of the server 
		keepalive_timeout 5; 
		client_max_body_size 4G;
	 	access_log /home/app-name/app-name/logs/nginx-access.log; 
		error_log /home/app-name/app-name/logs/nginx-error.log; 
		
		location /static/ { 
			alias /home/app-name/app-name/staticfiles/; 
		} 
		# checks for static file, if not found proxy to app 
		location / { 
			try_files $uri @proxy_to_app; 
		} 
		location @proxy_to_app { 
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
		proxy_set_header Host $http_host; 	
		proxy_redirect off; 
		proxy_pass http://app_server; 
		} 
	}

Create a symbolic link to the sites-enabled directory.	
```shell
sudo ln -s /etc/nginx/sites-available/app-name /etc/nginx/sites-enabled/app-name
```

Remove the default NGINX website.
```shell
sudo rm /etc/nginx/sites-enabled/default
```

Restart the NGINX service.
```shell
sudo service nginx restart
```

Any further code updates.	
```shell
sudo supervisorctl restart app-name	
```

## Configuring HTTPS Certificate

Protect the application with an HTTPS certificate provided by Lets Encrypt.
```shell
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt update
sudo apt install python-certbot-nginx
```

Install the certs.
```shell
sudo certbot –nginx
```

Choose option 2 to redirect all HTTP traffic to HTTPS.	
Set up the auto-renewal of the certs. Run the command below to edit the crontab file.
```shell
sudo crontab -e
```

Add the following line to the end of the file.
```shell
0 4 * * * /usr/bin/certbot renew –quiet
```

This command will run every day at 4 am. All certificates expiring within 30 days will automatically be renewed.
