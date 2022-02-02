# Self-hosted Docker Registry

1) Install Docker

2) Run `mkdir /var/lib/registry`

2) Run `docker run -d -p 5000:5000 -v /docker/registry:/var/lib/registry --restart always --name registry registry:2`

3) Use the following Nginx config:

```
server {
  listen 80;
  listen [::]:80;

  server_name registry.example.com;

  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  listen [::]:443 ssl;

  ssl_certificate /etc/letsencrypt/live/registry.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/registry.example.com/privkey.pem;

  server_name registry.example.com;

  location / {
    proxy_pass http://127.0.0.1:5000;
  }

  include snippets/letsencrypt.conf;
}
```
