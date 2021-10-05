# Open edX Platform
Open edX documentation for production environment on Ubuntu server 16.04

## Requierments
- Ubuntu 16.04 amd64
- Minimum 8GB of memory
- At least one 2.00GHz CPU or EC2 compute unit
- Minimum 25GB of free disk, 50GB recommended for production servers

## Setup Ubuntu before installation
Launch your Ubuntu 16.04 64-bit server and log in to it as a user that has full sudo privileges.

### Update your Ubuntu package sources:

```
sudo apt-get update -y
sudo apt-get upgrade -y
sudo reboot
```
### Configure /etc/hosts y /etc/hostname with your domain name and hostname
```
echo "127.0.0.1	localhost openedx.unam.mx" > /etc/hosts
echo "132.146.145.23	openedx openedx.unam.mx" >> /etc/hosts
echo "openedx.unam.mx" > /etc/hostname
```
### Automated installation with ansible
You choose the version of Open edX by setting the OPENEDX_RELEASE variable before running the commands. See Open edX Releases for the tags you can use (https://edx.readthedocs.io/projects/edx-developer-docs/en/latest/named_releases.html)

This script uses OPENEDX_RELEASE=hawthorn.master, feel free to change it to the one desired.
```
sudo wget https://raw.githubusercontent.com/CesarTavo/edxu-platform/master/edx.platform-install.sh
sudo chmod 755 edx.platform-install.sh
sudo nohup ./edx.platform-install.sh &
```
*The installation takes about 2 hours to complete*


## Check running services
```
sudo /edx/bin/supervisorctl status
```
You should see something like this
```
analytics_api                    RUNNING   pid 1858, uptime 17:12:34
certs                            RUNNING   pid 1860, uptime 17:12:34
cms                              RUNNING   pid 9876, uptime 13:37:31
discovery                        RUNNING   pid 1883, uptime 17:12:34
ecommerce                        RUNNING   pid 1866, uptime 17:12:34
ecomworker                       RUNNING   pid 1853, uptime 17:12:34
edxapp_worker:cms_default_1      RUNNING   pid 10006, uptime 13:37:23
edxapp_worker:cms_high_1         RUNNING   pid 10004, uptime 13:37:23
edxapp_worker:cms_low_1          RUNNING   pid 10002, uptime 13:37:23
edxapp_worker:lms_default_1      RUNNING   pid 10005, uptime 13:37:23
edxapp_worker:lms_high_1         RUNNING   pid 10008, uptime 13:37:23
edxapp_worker:lms_high_mem_1     RUNNING   pid 10003, uptime 13:37:23
edxapp_worker:lms_low_1          RUNNING   pid 10007, uptime 13:37:23
forum                            RUNNING   pid 1854, uptime 17:12:34
insights                         RUNNING   pid 1863, uptime 17:12:34
lms                              RUNNING   pid 9862, uptime 13:37:32
notifier-celery-workers          RUNNING   pid 1856, uptime 17:12:34
notifier-scheduler               RUNNING   pid 1861, uptime 17:12:34
xqueue                           RUNNING   pid 1852, uptime 17:12:34
xqueue_consumer                  RUNNING   pid 1859, uptime 17:12:34
```
## Rabbitmq service configuration
### Check if rabbitmq users where created on install for Open edX.

You should see three users with the following command
```
sudo rabbitmqctl list_users
```
- admin
- celery
- edx

In case the users where not created, we have to create them manually.
Take the user passwords from the file my-passwords.yml and insert them in the next commands.

**User admin** (RABBIT_ADMIN_PASSWORD)
```
sudo rabbitmqctl add_user admin YOUR-PASSWORD
sudo rabbitmqctl set_user_tags celery administrator
sudo rabbitmqctl set_permissions -p / celery ".*" ".*" ".*"
```
**User edx** (XQUEUE_RABBITMQ_PASS)
```
sudo rabbitmqctl add_user edx YOUR-PASSWORD
sudo rabbitmqctl set_user_tags edx administrator
sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```
**User celery** (EDXAPP_CELERY_PASSWORD)
```
sudo rabbitmqctl add_user celery YOUR-PASSWORD
sudo rabbitmqctl set_user_tags admin administrator
sudo rabbitmqctl set_permissions -p / edx ".*" ".*" ".*"
```
**Restart rabbitmq service**
```
sudo service rabbitmq-server restart
```

## Get starting

To get starting you have to change the variables `localhost` and `127.0.0.1`
to your domain name in `/edx/app/edxapp/cms.env.json` and
`/edx/app/edxapp/lms.env.json`

```
sed -i "s/localhost/DOMAIN/g" /edx/app/edxapp/cms.env.json
sed -i "s/localhost/DOMAIN/g" /edx/app/edxapp/lms.env.json
sed -i "s/127.0.0.1/DOMAIN/g" /edx/app/edxapp/cms.env.json
sed -i "s/127.0.0.1/DOMAIN/g" /edx/app/edxapp/lms.env.json
```

## Adding SUPERUSER OpenEdx

To add a superuser use this:

```
/edx/bin/python.edxapp /edx/bin/manage.edxapp lms manage_user mooc mooc@mooc.com  --staff --superuser --settings=aws
```
Where `mooc` is our superuser and `mooc@mooc.com` is its
email

To add or change its password use this:

```
sudo -u www-data /edx/bin/python.edxapp ./manage.py lms --settings aws changepassword mooc
```

It asks you the new password

## Mail Configuration

We use postfix to configure mail sending.

Check the next variables.

```
myhostname = openedx.unam.mx
mydomain = openedx.unam.mx

mydestination = $mydomain, openedx, localhost.localdomain, localhost

relayhost =

mynetworks = 127.0.0.0/8

inet_interfaces = all
```

restart the service

```
systemctl restart postfix
```

or with init.d

```
/etc/init.d/postfix restall
```

## Recomendations

To restart the processes more easily it is recommended to make a script that restarts the three services.

```
vi /edx/bin/restall
```

And add this lines

```bash
#!/bin/bash

sudo /edx/bin/supervisorctl restart lms
sudo /edx/bin/supervisorctl restart cms
sudo /edx/bin/supervisorctl restart edxapp_worker:
```

```
chmod +x /edx/bin/restall
chown www-data:root /edx/bin/restall
```

## Email configuration

Create a new user named `request` to receipt the new emails

```
useradd request
```

After that, you have to modify the common `cms`
in the route `/edx/app/edxapp/edx-platform/cms/envs/common.py`

In the **FEATURES** modify `'STUDIO_REQUEST_EMAIL'`

```
'STUDIO_REQUEST_EMAIL': 'request@openedx.unam.mx',
```

Now, if you create the TXT policy, our user can send and recieve emails.

But that is a problem, one user send and recieve itself the email
to solve that you have to forward its email to the administrador or administradors
to check the request.

In the **home** of **request** you have to create a file named `.forward`

Put the admin's emails in that file:

```
administrator1@mail.mx administrator2@mail.mx
```

## Install and Configure DKIM with Postfix

Please check out [this](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-dkim-with-postfix-on-debian-wheezy)

### Install OpenDKIM

```
sudo apt-get update
sudo apt-get dist-upgrade
```

Install OpenDKIM and it's dependencies:

```
sudo apt-get install opendkim opendkim-tools
```

### Configure OpenDKIM

Configure the main file:

```
vi /etc/opendkim.conf
```

Check the variables with the `opendkim.conf` file

Connect the milter to Postfix:

```
vi /etc/default/opendkim
```

Configure postfix to use this milter:

```
vi /etc/postfix/main.cf
```

Make sure that these two lines are present in the Postfix config file and are not commented out:

```
milter_protocol = 2
milter_default_action = accept
```

**AND PLEASE, FOLLOW THIS MANUAL**

[OPEN DKIM](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-dkim-with-postfix-on-debian-wheezy)

## Changing the default language

Change variables in `/edx/app/edxapp/edx-platform/lms/envs`

AND

If you want to change the default language from the web page
only change the next variables in the `/edx/app/edxapp/cms.env.json`
and `/edx/app/edxapp/lms.env.json`

```
"LANGUAGE_CODE": "es-419",
```

If you want to know what languages you have
please make `ls /edx/app/edxapp/edx-platform/conf/locale/` to know it.

## Changing Time Zone

If you want to change the default time zone just change next variable
in the `/edx/app/edxapp/cms.env.json` and `/edx/app/edxapp/lms.env.json`

```
"TIME_ZONE": "America/Mexico_City"
```
## Configuring SSL with Let´s Encrypt for cms and lms
Update packages and install certbot
```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
```
### LMS nginx configuration
Edit */etc/nginx/sites-enabled/lms*

Change this
```
listen 80 default_server;
```
To this and add server_name
```
listen 443 ssl;
server_name openedx.unam.mx
```

### CMS nginx configuration
Edit */etc/nginx/sites-enabled/cms*

Change this
```
listen 80 default_server;
```
To this and add server_name
```
listen 443 ssl;
server_name studio.openedx.unam.mx;
```

### Install certificates for both domains
```
sudo certbot --authenticator standalone --installer nginx --pre-hook "service nginx stop" --post-hook "service nginx start"
```
You should see similar configuration lines in both cms and lms nginx files
```
ssl_certificate     /etc/letsencrypt/live/openedx.unam.mx/fullchain.pem; # managed by Certbot
ssl_certificate_key /etc/letsencrypt/live/openedx.unam.mx/privkey.pem; # managed by Certbot
include             /etc/letsencrypt/options-ssl-nginx.conf;
ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;
```
*Ready! Certificates are installed!*

## Theme configuration
