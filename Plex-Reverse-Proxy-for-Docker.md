# Setting up a Plex reverse proxy to connect to a Dockerized instance of Plex Media Server

This guide is intended to explain the necessary configuration to set up an Nginx-based reverse proxy server to offer connectivity to your Plex Media Server running in a Docker container.

## Why would you want to do this?

There are several reasons:

* **Use your Plex media server completely independent of a Plex account.** When you run Plex in Docker, it can be challenging to connect to it from your own network, largely due to how Plex detects whether your connection is "local" or "remote". The Plex web UI will redirect you to Plex to sign in if the Plex server does not consider you to be a local connection. A reverse proxy lets you forward the correct headers to cause Plex to always interpret your connection as local.
* **Manage SSL on your own.** By installing your own SSL certificate into the reverse proxy, you are in control of SSL connectivity to your Plex server. Since mobile and TV apps do not currently support manual connections to a secured Plex server, you can also set up your proxy to allow non-secured access from your LAN for such devices, again avoiding the need to sign in to Plex.
* **Logging.** Your reverse proxy will generate logs of all accesses to Plex. While these logs won't be as detailed as the Plex server logs, they will be in a standard format that can be parsed by tools such as [The Webalizer](http://www.webalizer.org) or other dashboard systems. 
* **Remotely access your Plex server privately and securely using a VPN.** If you have a VPN which allows you into your home LAN, your reverse proxy will function over that VPN just as any other service on your LAN would. This means you can securely access your Plex content from anywhere as long as you are on your VPN. 
    * Note that you will need to manually configre your clients to limit bandwidth usage if you need to limit it due to available upstream bandwidth; clients will interpret your connection as "local" and will try to stream at full bandwidth over the VPN link.
* **You can secure Plex behind the Docker network, leaving the proxy as the only gateway into Plex from your LAN.** No need to forward ports or use `--net host`. You can even implement IP blacklisting or any other rule nginx is capable of following to restrict access to Plex.

## Getting Started

If you already have a Plex instance running in Docker, you'll need to destroy it and create a new one. If you followed the instructions for setting up Plex on Docker, you performed a volume mount for multiple locations:

* **/transcode**: Transcoding temp folder
* **/config**: Configuration
* **/data**: Default location for media

When you re-create your Docker container, you'll need to use these same mounts. If you do this right, your new Plex container will automatically pick up on your old settings and Plex will ultimately come back up just as if you hadn't changed anything.

### If you forgot to create volume mounts for /transcode or /config:

It's not impossible to recover from this, but it is a bit tricky. 

**Method 1**

This method assumes you at least created a volume mount for your media. If this is the case, you have one easy way out of your container - through this mount. You can use **docker exec** to connect to your container and obtain a shell, and copy the files to your media folder temporarily:

    host:# docker exec -it <name of your Plex container> /bin/sh
    container:# mkdir /data/server-data
    container:# cp -av /config /transcode /data/server-data
    container:# exit

After you do this, **stop your Docker container right away.** This prevents anything else from happening before you migrate to a new container:

    host:# docker stop <name of your Plex container>

**Method 2**

If you didn't use *any* mounts at all, you are in a bit of a tougher spot. But you still have an option:

1. Get a shell within the container.

        host:# docker exec -it <name of your Plex container> /bin/sh

1. Make an archive of the contents of the `/config` and `/transcode` folders:

        container:# tar cf /backup.tar /config /transcode

1. Exit the container

        container:# exit

1. Copy the archive out of the container

        host:# docker cp <name of your Plex container>:/backup.tar .

You now have a copy of your configuration. You can extract this tarfile somewhere on your system and next time bind-mount it into the correct locations.

## Creating a new Plex Docker container

For this example, I will assume your folders are as follows (please change the commands to point to your actual folders):

* Configuration folder: /data/plex/config
* Transcode folder: /data/plex/transcode
* Media folder: /media

1. Create a new virtual network to run your Plex container on.

        host:# docker network create plexnet

    (You can name the network anything you like, but it must be the same in all commands.)

1. Start a Plex docker container in this network, making the correct bind mounts:

        host:# docker run -d \
            -v /data/plex/config:/config \
            -v /data/plex/transcode:/transcode \
            -v /media:/data \
            --net plexnet \
            --name plex
            linuxserver.io/plex

Plex should now be up and running, however you can't access it yet. This is by design - we didn't forward any ports at all using `-p` on the Plex container, as we don't need to - that will be handled by the proxy server.

## Creating the Plex proxy container

Now we're going to create and configure an `nginx` container which will run as our Plex proxy server.

1. Create an nginx container which will serve as our reverse proxy (but don't start it yet!)

        host:# docker create -d \
            -p 32400:32400 \
            -p 32469:32469 \
            -v /data/plex/proxylogs:/var/log/nginx \
            --net plexnet \
            --name plex-proxy \
            nginx:alpine

    A few things to note here:

    * The ports 32400 and 32469 are the default Plex ports for insecure and secure connections, respectively. The nginx configuration we will provide also provides the proxy server on these ports. If you want to expose your Plex server on different ports, you can do it in the `-p` flags by changing the first number. For example, if you wanted to run your Plex server on standard HTTP ports, you could use `-p 80:32400 -p 443:32469`.

    * The `-v` option puts nginx's logs somewhere on your host platform. This is optional, but it is handy if you want to be able to access and parse the proxy logs for analysis or dashboard interfaces. 

    * The `--net` network name must match the one you created in the `docker network create` command above.

1. Create a directory on your server to use as a working directory for the configuration.

1. Now, create a file called `nginx.conf` and add the following contents:

        worker_processes  1;
        error_log  /var/log/nginx/error.log;

        events {
            worker_connections  1024;
        }

        http {

            ssl_session_cache shared:SSL:10m;
            ssl_session_timeout 10m;

            upstream plex_backend {
                server plex:32400;
                keepalive 32;
            }

            server {
                listen 32469 ssl http2;
                listen 32400;

                server_name !!YOUR_SERVERS_DNS_NAME_OR_IP_ADDRESS_HERE!!;
                send_timeout 100m; 
                
                # Setup SSL certificate
                ssl_certificate /etc/nginx/server.crt;
                ssl_certificate_key /etc/nginx/server.key;
                
                # Allow downgrade SSL security.
                # This is done because some clients require older SSL security to work properly.
                # Newer browsers supporting the most recent TLS versions will work fine either way.
                ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
                ssl_prefer_server_ciphers on;
                ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:ECDHE-RSA-DES-CBC3-SHA:ECDHE-ECDSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
                ssl_session_tickets off;

                # DH parameters file.
                ssl_dhparam /etc/nginx/dhparam.pem;
                ssl_ecdh_curve secp384r1;

                # Plex has a lot of text script which is easily compressed.
                # If these settings cause playback issues with devices, remove them. (Haven't encountered any yet)
                gzip on;
                gzip_vary on;
                gzip_min_length 1000;
                gzip_proxied any;
                gzip_types text/plain text/css text/xml application/xml text/javascript application/x-javascript image/svg+xml;
                gzip_disable "MSIE [1-6]\.";

                # nginx default client_max_body_size is 1MB, which breaks Camera Upload feature from phones.
                # Increasing the limit fixes the issue.
                # Note if you are sending VERY LARGE files (e.g. 4k videos) you will need to increase this much further.
                client_max_body_size 100M;

                # Set headers for Plex server.
                proxy_http_version 1.1;
                proxy_set_header Host localhost; # Forces Plex to see all connections from the proxy as local
                proxy_set_header Referer localhost; # Forces Plex to see all connections from the proxy as local
                proxy_set_header Origin $scheme://localhost:$server_port; # Forces Plex to see all connections from the proxy as local
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_set_header Accept-Encoding ""; # Disables compression between Plex and Nginx

                # Disable buffering - send to the client as soon as the data is received from Plex.
                proxy_redirect off;
                proxy_buffering off;

                location / {
                    proxy_pass http://plex_backend;
                }
            }
        }

    There's a lot going on here, but this configuration as is should work fine for you. 

    Many of the more advanced options are documented. Here are a few extra valuable pointers:

    * **Make sure you change the `server_name` parameter to match the address you expect to access your server at.** You can either enter a hostname or an IP address - basically, enter here exactly what you expect to put into a browser to access the server. Also make sure the terminating `;` remains.
    * If you really don't want SSL security at all, you can exclude everything starting with `ssl_`. You must then also ensure not to include `listen 32469 ssl http2`, as nginx will fail to start if asked to listen on SSL without the necessary parameters.
    * I advise you to leave the ports in this config as they are, and use the docker `-p` switch to change the ports exposed to the network. 
    * Notice how in the `upstream` block the server address is listed as just `plex`. This name must match the name given in the `--name` option for the *Plex container* you started first. Docker provides its own internal DNS resolution that allows containers on one network to communicate with each other by using their container names as if they were hostnames. 

1. If you are intending to use SSL, generate am appropriate SSL certificate and key and name them `server.crt` and `server.key`. Put them in the same directory as your `nginx.conf` file.

1. If you are using SSL you also need to generate DH parameters using this command: `openssl dhparam -out dhparam.pem 2048`

1. You're now ready to move these files into your Docker container:

    * `docker cp nginx.conf plex-proxy:/etc/nginx` 
    * `docker cp server.crt plex-proxy:/etc/nginx` (if using SSL)
    * `docker cp server.key plex-proxy:/etc/nginx` (if using SSL)
    * `docker cp dhparam.pem plex-proxy:/etc/nginx` (if using SSL)

1. Now, go ahead and start your proxy!

        docker start plex-proxy

1. Finally, try to access your Plex server at the address of your proxy, for example `https://my-proxy.local.lan:32469/web/index.html` If you see the Plex web interface, you're all set!

