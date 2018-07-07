# django-postgresql-nginx-supervisor-gunicorn-deploy-script

```bash
#!/bin/bash

export DIRECTORY=`pwd`
export CURRENT_USER=`whoami`
export APPLICATION_NAME=


###### DATABASE CONFIGS ######
export POSTGRESQL_DB_NAME=
export POSTGRESQL_DB_USERNAME=
export POSTGRESQL_DB_PASSWORD=

APP_ADMIN_USERNAME=
APP_ADMIN_EMAIL=
APP_ADMIN_PASSWORD=

GUNICORN_WORKER_NUMBER=


## create logs folder for application and web server logs
mkdir logs

## install all requirements
apt-get update
apt-get -y dist-upgrade
apt-get install -y supervisor nginx
apt-get install -y python-pip python-dev postgresql postgresql-server-dev-all postgresql-contrib
apt-get install -y libtiff5-dev libjpeg8-dev zlib1g-dev \
    libfreetype6-dev liblcms2-dev libwebp-dev libharfbuzz-dev libfribidi-dev \
    tcl8.6-dev tk8.6-dev python-tk
pip install -r requirements.txt


### Create postgresql user database and set privileges
su postgres bash -c "psql -c \"CREATE database $POSTGRESQL_DB_NAME;\""
echo "\n db created. \n"
su postgres bash -c "psql -c \"CREATE USER $POSTGRESQL_DB_USERNAME WITH PASSWORD '$POSTGRESQL_DB_PASSWORD';\""
echo "\n db user created \n"
su postgres bash -c "psql -c \"GRANT ALL PRIVILEGES ON DATABASE $POSTGRESQL_DB_NAME to $POSTGRESQL_DB_USERNAME;\""
echo "\n db privileges added."

#### make migrations and migrate db also create super user for web app
python manage.py makemigrations
echo "\n db makemigrations"
python manage.py migrate
echo "\n db migrate"
python manage.py shell -c "from django.contrib.auth.models import User; User.objects.create_superuser('$APP_ADMIN_USERNAME', '$APP_ADMIN_EMAIL', '$APP_ADMIN_PASSWORD')"
echo "\n db create super user completed."


### create gunicorn script
echo "
#!/bin/bash

NAME=$APPLICATION_NAME
DJANGODIR=$DIRECTORY
SOCKFILE=$DIRECTORY/run/gunicorn.sock
NUM_WORKERS=$GUNICORN_WORKER_NUMBER
DJANGO_SETTINGS_MODULE=gop.settings
DJANGO_WSGI_MODULE=gop.wsgi

exec gunicorn $DJANGO_WSGI_MODULE:application \
  --name $APPLICATION_NAME \
  --workers $NUM_WORKERS \
  --user=$CURRENT_USER \
  --bind=127.0.0.1:8001\
  --log-level=info" > gunicorn.sh
echo "\n gunicorn script added."

sudo sh -c "echo
\"[program:$APPLICATION_NAME]
command = sh $DIRECTORY/gunicorn.sh
user = $CURRENT_USER
stdout_logfile = $DIRECTORY/logs/gunicorn.log
redirect_stderr = true
environment=LANG=en_US.UTF-8,LC_ALL=en_US.UTF-8\" > /etc/supervisor/conf.d/app.conf"

echo "\n supervisor script added."

supervisorctl reread
supervisorctl update
supervisorctl restart all
rm -rf /etc/nginx/sites-available/default

echo "\n supervisor reread , update, restart all complated."

sudo sh -c "echo \" server {
    listen   80;
    client_max_body_size 4G;
    access_log $DIRECTORY/logs/nginx-access.log;
    error_log $DIRECTORY/logs/nginx-error.log;
    location /static/ {
        alias   $DIRECTORY/static/;
    }
    location /media/ {
        alias   $DIRECTORY/media/;
    }
    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header Host $http_host;
        proxy_redirect off;
        if (!-f $request_filename) {
            proxy_pass http://127.0.0.1:8001;
            break;
        }
    }
}\" > /etc/nginx/sites-available/app"

echo "\n nginx config added."
sudo ln -s /etc/nginx/sites-available/gop /etc/nginx/sites-enabled/gop

sudo service nginx restart
echo "\n nginx restarted."
```
