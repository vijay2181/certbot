# certbot

## How Let’s Encrypt Works

Before issuing a certificate, Let’s Encrypt validates ownership of your domain. The Let’s Encrypt client, running on your host, creates a temporary file (a token) with the required information in it. The Let’s Encrypt validation server then makes an HTTP request to retrieve the file and validates the token, which verifies that the DNS record for your domain resolves to the server running the Let’s Encrypt client.
- certbot is Lets Encrypt client.
- certbot can automatically configure NGINX for SSL/TLS. It looks for and modifies the server block in your NGINX configuration that contains a server_name directive with the domain name you’re requesting a certificate for. In our example, the domain is www.example.com.
- certbot needs compitable python version for installation
- by using certbot we can install certs on specific ip address for the domain, where we need have dns record pointed for ip
- Create a DNS record that associates your domain name and your server’s public IP address.
- certs are installed for ip address
- python 3.9 version comes default for amamzon linux 2023
- AmiId: ami-01450e8988a4e7f44
- AmiLocation: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64
- Nginx is acting as a reverse proxy for the application. A reverse proxy is a server that sits between client devices (like web browsers) and a backend server (like your Docker container running the application). The reverse proxy forwards client requests to the backend server and then returns the server's responses to the clients.
- Let's Encrypt has a rate limit of 5 certificates per set of domains per 7 days.The error message suggests retrying after 2023-12-29T16:06:33Z, which is the time when the rate limit for this set of domains will be reset. You can wait until that time and then try again.

```
python3.9 installation 
======================
cd /usr/src
wget https://www.python.org/ftp/python/3.9.10/Python-3.9.10.tgz
tar xzf Python-3.9.10.tgz
cd Python-3.9.10
yum install gcc openssl-devel bzip2-devel libffi-devel zlib-devel -y
./configure --enable-optimizations
make altinstall
rm -f /usr/bin/python3   (removes any previous python3 soft links)
ln -s /usr/local/bin/python3.9 /usr/bin/python3
python3 --version
yum install python3-pip -y
```


```
certbot setup:
==============
cd /root
yum install git -y
git clone https://github.com/certbot/certbot
cd certbot
pip3 install virtualenv
sudo yum install augeas augeas-libs -y
tools/venv.py
. venv/bin/activate
certbot certonly -n --agree-tos --dns-route53 \
  -d vijay.com --dns-route53-propagation-seconds 60 \
  --email vijay@gmail.com
```

```
nginx setup:
============
yum install nginx -y
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
```

```
vi /etc/nginx/nginx.conf

user  nginx;
worker_processes  1;
error_log  /var/log/nginx/error.log;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    include /etc/nginx/conf.d/*.conf;
    index   index.html index.htm;

    server {
        listen 80;
        server_name vijay.com;    #your domain 
        return 301 https://$host$request_uri;
    }
    server {
        listen 443;
        server_name vijay.com;

        ssl_certificate           /etc/letsencrypt/live/vijay.com/fullchain.pem;
        ssl_certificate_key       /etc/letsencrypt/live/vijay.com/privkey.pem;

        ssl on;
        ssl_session_cache  builtin:1000  shared:SSL:10m;
        ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_ciphers "ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS";

        location / {
            proxy_set_header        Host $host;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header        X-Forwarded-Proto $scheme;

            # Fix the “It appears that your reverse proxy set up is broken" error.
            proxy_pass          http://127.0.0.1:5000;   #application port for ngnix as reverse proxy(app will be accessed on 5000 and redirects to 80)
            #proxy_read_timeout  90;
            proxy_connect_timeout 600;
            proxy_send_timeout 600;
            proxy_read_timeout 600;
            send_timeout 600;
        }
    }
}

systemctl enable nginx
sudo systemctl start nginx
```

- Traffic comes to the EC2 instance on ports 80 and 443.
- Nginx, which is configured to listen on ports 80 and 443, acts as a reverse proxy. It takes the incoming traffic and forwards it to the Docker container running the application on port 5000.
- The Docker container is where your application is running and is accessible internally on port 5000.
