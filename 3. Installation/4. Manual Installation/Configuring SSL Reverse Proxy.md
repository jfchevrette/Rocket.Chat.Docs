# Configuring SSL Reverse Proxy

Rocket.Chat is a middle tier application server, by itself it does not handle SSL.   However, Rocket.Chat works well with several industrial grade, battle-tested reverse proxy servers (see Nginx below, for example) that you can configure to handle SSL.

**Note:** You must use the outside https address for the value at ```ROOT_URL``` in [[Section 3|Deploy-Rocket.Chat-without-docker#3-download-rocketchat]] above.  This includes the "https://" and leave off the port number.  So instead of ```ROOT_URL=http://localhost:3000``` use something like ```https://your_hostname.com```

## Running behind a Nginx SSL Reverse Proxy

**Note:** These instructions were written for Ubuntu.  For Amazon Linux, the conf file for the proxy goes in `/etc/nginx/conf.d/` and needs to have a discrete name ending in `.conf` and Nginx is installed using `yum -y install nginx`.

Run this as root:

```shell
apt-get install nginx
```

Add your private key to ```/etc/nginx/certificate.key```

Lock down permissions: ```chmod 400 /etc/nginx/certificate.key```

Add your certificate to ```/etc/nginx/certificate.crt```

Edit ```/etc/nginx/sites-enabled/default``` or if you use nginx from docker ```/etc/nginx/conf.d/default.conf``` and be sure to use your actual hostname in lieu of the sample hostname "your_hostname.com" below.

```
# Upstreams
upstream backend {
    server 127.0.0.1:3000;
}

# HTTPS Server
server {
    listen 443;
    server_name your_hostname.com;

    error_log /var/log/nginx/rocketchat.access.log;

    ssl on;
    ssl_certificate /etc/nginx/certificate.crt;
    ssl_certificate_key /etc/nginx/certificate.key;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # don’t use SSLv3 ref: POODLE

    location / {
        proxy_pass http://backend/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forward-Proto http;
        proxy_set_header X-Nginx-Proxy true;

        proxy_redirect off;
    }
}
```

Restart Nginx: ```service nginx restart```

## Running under Plesk Onyx behind Nginx

Plesk Onyx has now a docker installation and Nginx proxy docker rule generator builtin, that doesn't support adding custom directives. Disable it and add the rules manually in the additional Nginx directives space. A scheme follows (replace 30000 with your external docker mapped port).

```
#manual extension docker with socket upgrade begin
location ~ ^/.* {
	proxy_pass http://0.0.0.0:30000;
	proxy_set_header Host             $host;
	proxy_set_header X-Real-IP        $remote_addr;
	proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;

	proxy_set_header Upgrade $http_upgrade;
	proxy_set_header Connection "upgrade";
	proxy_set_header X-Forward-Proto http;
	proxy_set_header X-Nginx-Proxy true;
	proxy_http_version 1.1;

	proxy_redirect off;
}

#extension docker end
```

## Running behind an Apache SSL Reverse Proxy

**Note:** You must use the outside https address for the value at ```ROOT_URL``` in [[Section 3|Deploy-Rocket.Chat-without-docker#3-download-rocketchat]] above.  This includes the "https://" and leave off the port number.  So instead of ```ROOT_URL=http://localhost:3000``` use something like ```https://your_hostname.com```

Run this as root:

```shell
apt-get update
apt-get install apache2
a2enmod proxy_http
a2enmod proxy
a2enmod ssl
a2enmod proxy_wstunnel
```

Add your private key to ```/etc/ssl/private/chat.domain.com.key```

Lock down permissions: ```chmod 400 /etc/ssl/private/chat.domain.com.key```

Add your certificate to ```/etc/ssl/certs/chat.domain.com.crt```

Add your intermediate to ```/etc/ssl/certs/intermediate.ca.pem```

Edit ```/etc/apache2/sites-enabled/rocketchat.conf``` and be sure to use your actual hostname in lieu of the sample hostname "your_hostname.com" below.

```
<VirtualHost *:443>

ServerAdmin it@domain.com
ServerName chat.domain.com
ErrorLog /var/log/chat.domain.com_error.log
TransferLog /var/log/chat.domain.com_access.log
LogLevel info
SSLEngine On
SSLCertificateFile /etc/ssl/certs/chat.domain.com.crt
SSLCertificateKeyFile /etc/ssl/private/chat.domain.com.key
SSLCertificateChainFile /etc/ssl/certs/intermediate.ca.pem

<Location />
        Order allow,deny
        Allow from all
</Location>

RewriteEngine On
RewriteCond %{HTTP:Upgrade} =websocket [NC]
RewriteRule /(.*)           ws://localhost:3000/$1 [P,L]
RewriteCond %{HTTP:Upgrade} !=websocket [NC]
RewriteRule /(.*)           http://localhost:3000/$1 [P,L]

ProxyPassReverse / http://localhost:3000/
</VirtualHost>
```

Restart Apache: ```service apache2 restart```

## Running behind a Caddy Reverse Proxy with Free SSL

First, download [Caddy](https://caddyserver.com/)

`curl https://getcaddy.com | bash`

Now Caddy is installed, but you still need a service to run Caddy http server on the background.

You can find services backed by the community [here](https://github.com/mholt/caddy/tree/master/dist/init)

You must have at least the port **443** opened so the Caddy server will request an SSL certificate from Let's Encrypt

You can also open the port 80 to redirect http requests to https.

Open `/etc/caddy/Caddyfile`

Insert

```
your_domain.com {
        proxy / 127.0.0.1:3000 {
                proxy_header X-Forwarded-Proto {scheme}
                proxy_header X-Forwarded-For {host}
                proxy_header Host {host}
                websocket
        }
}
```
