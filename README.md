Pastefile           [![Build Status](https://travis-ci.org/guits/pastefile.svg?branch=master)](https://travis-ci.org/guits/pastefile)
=========

Little daemon written with python flask for sharing any files quickly via http
------------------------------------------------------------------------------

- [Installation](#Installation)
  - [Quick run for test purpose](#Quick-run-for-test-purpose)
  - [Standard](#Standard)
  - [Docker](#Docker)
- [Options](#Options)
- [Usage](#Usage)


# Installation
You can either install by yourself with nginx or apache and use a custom configuration for uwsgi or build a docker image with the Dockerfile provided.


## Quick run for test purpose

If you just want to test pastefile quickly, we provide `pastefile-run.py` script only for test purpose.

```bash
apt-get install -y git python-dev python-pip
pip install -r https://raw.githubusercontent.com/guits/pastefile/master/requirements.txt
git clone https://github.com/pastefile/pastefile.git
cd pastefile && cp pastefile.cfg.sample pastefile.cfg
# Modify pastefile.cfg config to adapt pastefile directories
./pastefile-run.py -c $PWD/pastefile.cfg
```

## Standard
```bash
apt-get install -y git nginx-full python-pip python-dev uwsgi-plugin-python uwsgi
pip install -r https://raw.githubusercontent.com/guits/pastefile/master/requirements.txt
```

```bash
git clone https://github.com/pastefile/pastefile.git /var/www/pastefile
```

> **note** that ```/var/www/pastefile``` must be writable by the uwsgi process that will be launched later. You may have to ```chown <uid>:<gid>``` it with right user/group.

Write the configuration file:

```bash
curl -s -o/etc/pastefile.cfg  https://raw.githubusercontent.com/guits/pastefile/master/pastefile.cfg.sample
```

Change parameters as you need:

```bash
vim /etc/pastefile.cfg
```
**Nginx configuration:**

> /etc/nginx/nginx.conf :

```
user www-data;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
  
events {
    worker_connections 1024;
}
  
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
  
    access_log  /var/log/nginx/access.log  main;
  
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   1;
    types_hash_max_size 2048;
  
    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
  
    include /etc/nginx/conf.d/*.conf;
}
```


> /etc/nginx/conf.d/pastefile.conf :
  
```
server {
    listen 80 default_server;


        location / { try_files $uri @pastefile; }

        location @pastefile {
            include uwsgi_params;
            uwsgi_pass unix:///tmp/pastefile.sock;
        }
}
```

**uwsgi configuration:**

> /etc/uwsgi/apps-available/pastefile.ini :

```
[uwsgi]
socket = /tmp/pastefile.sock
module = pastefile.app:app
chdir  = /var/www/pastefile
uid = 33
gid = 33
env = PASTEFILE_SETTINGS=/etc/pastefile.cfg
processes = 1
threads = 1
```

Enable it with:
```ln -s /etc/uwsgi/apps-available/pastefile.ini /etc/uwsgi/apps-enabled/pastefile.ini```

Now, you just have to launch nginx and uwsgi via systemd:

```bash
systemctl start nginx.service uwsgi.service
```


## Docker

First, clone the repo where you want:
```bash
git clone https://github.com/pastefile/pastefile.git
```
Build the image:
```bash
docker build --rm -t pastefile ./
```
You can then run a container:
```bash
docker run -d --name=pastefile pastefile
```
this is the easiest way to get a pastefile application running quickly.


# Options

|Parameters              | Usage                                                                                                                                                                                       |
|------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|UPLOAD_FOLDER           | Where the files are stored.                                                                                                                                                                 |
|FILE_LIST               | The file that act as the db (jsondb)                                                                                                                                                        |
|TMP_FOLDER              | The folder where the files are stored during the transfer                                                                                                                                   |
|EXPIRE                  | How many long the files are stored (in seconds)                                                                                                                                             |
|DEBUG_PORT              | The port used for debugging mode                                                                                                                                                            |
|LOG                     | The path to the log file                                                                                                                                                                    |
|DISABLED_FEATURE        | A comma separated list of features you want to disable. Allowed value : `delete`, `ls`                                                                                                                        |
|DISPLAY_FOR             | A comma separated list of browsers to display file like png or txt directly instead of asking for download. Allowed list from flask `request.user_agent.browser`                                                  |
|UWSGI_CONNECT_TIMEOUT   | Defines a timeout for establishing a connection with a uwsgi server. Default 60s                                                                                                            |
|UWSGI_READ_TIMEOUT      | Defines a timeout for reading a response from the uwsgi server. The timeout is set only between two successive read operations, not for the transmission of the whole response. Default 60s |
|UWSGI_SEND_TIMEOUT      | Sets a timeout for transmitting a request to the uwsgi server. The timeout is set only between two successive write operations, not for the transmission of the whole request. Default 60s  |

> **Note**:

> **The directory and the db file must be writable by uwsgi.**

> the format must be KEY = VALUE.

> the KEY must be in uppercase

> if the parameter is a string, you must quote it with **""**


> **Optimize** : Pastefile use `shutil.move`. To improve performance we recommand you to use the same filesystem for `TMP_FOLDER` and `UPLOAD_FOLDER` to allow rename file instead of file copy.

# Usage
Upload a file:
```bash
curl -F file=@</path/to/the/file> http://pastefile.fr
```

Upload a file with burn after read :
```bash
curl -F file=@</path/to/the/file> -F burn=true http://pastefile.fr
```

View all uploaded files:
```bash
curl http://pastefile.fr/ls
```

Get infos about one file:
```bash
curl http://pastefile.fr/<id>/infos
```

Get a file:
```bash
curl -JO http://pastefile.fr/<id>
```

Delete a file:
```bash
curl -XDELETE http://pastefile.fr/<id>
```

You can use this tips by adding this line in your ```.bashrc``` :
```bash
pastefile() { curl -F file=@"$1" http://pastefile.fr; }
```
so you can just type:
```bash
pastefile /my/file
```
to easily upload a file.


# Extra

Simple script to take a screenshot on a selected region of the screen and then upload the screenshot on pastefile.
The pastefile link will be automatically copy in your copy/paste buffer.


```bash
#!/bin/bash

# require :
# apt-get install imagemagick xsel libnotify-bin

filename=$(mktemp --suffix=_screenshot.png)
# Take the screenshot
import $filename

# Upload the file on pastefile
url=$(curl -F "file=@${filename}" http://pastefile.fr)
if [ "$?" = "0" ]; then
    notify-send "image uploaded! $url"

    # Add file to all clipboard
    echo -n "$url"|xsel -i -p
    echo -n "$url"|xsel -i -s
    echo -n "$url"|xsel -i -b
    echo "$url" >> /tmp/import.log
else
    notify-send --urgency=critical "Upload failed."
fi
```
