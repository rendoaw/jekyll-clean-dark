---
layout: post
comments: true
title: "Migrate Nextcloud from VM to Docker"
categories: blog
descriptions: Migrate Nextcloud from VM to Docker in Simple Way
tags: 
  - docker
  - nextcloud
  - nginx
date: 2018-04-25T10:39:55-04:01
---


* Note:
    * This is my personal nextcloud, so i only have one user and i don't bother to backup the setting/configuration. I only care about the files.


## What i have

* I have an Nextcloud (derivative of owncloud) hosted in one of VPS provider under domain let say, cloud.mydomain.com
* This nextcloud service has public SSL certificate from let's encrypt.

## What i want 

* I want to combined the nextcloud service into one of my other server
* The target server already have several web service which is behind nginx reverse proxy. 
* This nextcloud service should be put behind nginx as well



## Migration steps

* Find out where is the nextcloud data folder from the old server

    ```
    # cd /var/www/html/config
    # cat config.php | grep datadirectory
    'datadirectory' => '/data/www',
    ```

    * ok, i need to backup the files from old server /data/www


* copy the data from old server to new server

    ```
    # mkdir -p /data
    # mkdir -p /data/docker
    # mkdir -p /data/nextcloud
    # mkdir -p /data/nextcloud/www
    # cd /data/docker/nextcloud/www
    # scp -rp user@oldserver:/data/www/* .
    ```

    * note: extra www subdirectory here to make sure i can do chown from inside container. 

* copy the existing SSL certificates

    ```
    # scp -rp root@oldserver:/etc/letsencrypt/live/cloud.mydomain.com /etc/letsencrypt/live/
    ```


* On the new server, launch a nextcloud docker image which is available from docker hub. For this case, i want to store the data on the host server, not inside the container, 

    ```
    # docker run -d -it --name nextcloud --restart=always -v /data/docker/nextcloud:/data -p 10080:80 nextcloud
    ```

    * i will put my data on the host server at /data/docker/nextcloud.
    * the container name from docker hub is "nextcloud"
    * I am naming my container as "nextcloud" as well
    * I mount /data/docker/nextcloud as /data inside the container
    * i use restart=always to make sure the container is restarted if the host is rebooted
    * I am exposing nextcloud UI from port 80 inside container to host port 10080



* Attach to the container, and fix the permission

    ```
    rendo@newserver# docker exec -it nextcloud /bin/sh
    # cd /data
    # ls -al
    total 12
    drwxrwxrwx  3 root root 4096 Apr 26 02:37 .
    drwxr-xr-x 82 root root 4096 Apr 26 02:37 ..
    drwxr-xr-x  4 root root 4096 Apr 26 02:37 www
    # chown -R www-data www
    # chmod -R 0770 www
    ```


* Attach to the container, and add cloud.mydomain.com to trusted domain

    ```
    rendo@newserver# docker exec -it nextcloud /bin/sh
    # cd /var/www/html
    # busybox vi config/config.php

    modify trusted domain part so it looks like:

        'trusted_domains' =>
        array (
            0 => '192.168.100.17:10080',
            1 => 'cloud.mydomain.com',
        ),
    ```


* Connect to https://newserver:10080 and re-setup nextcloud
    * set admin username/password
    * Since this is personal nextcloud, sqllite is enough for me
    * Set data directory to /data/www


* Try to login to https://newserver:10080 and via see if i can see all the files


* On new server Configure nginx to redirect https://cloud.mydomain.com to http://localhost:10080

    ```
    # vim /etc/nginx/sites-enabled/default

    add something like this:

    server {
        listen 443 ssl;
        server_name cloud.mydomain.com;

        root html;
        index index.html index.htm;

        ssl on;
        ssl_session_timeout 5m;

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
        ssl_prefer_server_ciphers on;
        ssl_dhparam /etc/nginx/ssl/dhparam.pem;
        ssl_certificate /etc/letsencrypt/live/cloud.mydomain.com/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/cloud.mydomain.com/privkey.pem; # managed by Certbot

        location / {
            proxy_pass_header Authorization;
            proxy_pass http://localhost:10080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP  $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_buffering off;
            client_max_body_size 0;
            proxy_read_timeout  36000s;
            proxy_redirect off;
            proxy_ssl_session_reuse off;
        }


    }
    ```


* Restart nginx

    ```
    # /etc/init.d/nginx restart
    ```


* Update the DNS to point cloud.mydomain.com to new server IP


* Wait until DNS changes is propagated


* Try to open the nextcloud using the old domain https://cloud.mydomain.com


That's it !


