# Self-hosted Docker Registry

1) Install Docker: https://docs.docker.com/engine/install/debian/

2) Run `docker run -d -p 5000:5000 -v /docker/registry:/var/lib/registry --restart always --name registry registry:2`

3) Use the following Nginx config:

```
server {
  listen 80;
  listen [::]:80;

  server_name registry.example.com;

  return 301 https://$host$request_uri;
}

## Set a variable to help us decide if we need to add the
## 'Docker-Distribution-Api-Version' header.
## The registry always sets this header.
## In the case of nginx performing auth, the header is unset
## since nginx is auth-ing before proxying.
map $upstream_http_docker_distribution_api_version $docker_distribution_api_version {
  '' 'registry/2.0';
}

server {
  listen 443 ssl;
  listen [::]:443 ssl;

  # disable any limits to avoid HTTP 413 for large image uploads
  client_max_body_size 0;

  # required to avoid HTTP 411: see Issue #1486 (https://github.com/moby/moby/issues/1486)
  chunked_transfer_encoding on;

  ssl_certificate /etc/letsencrypt/live/registry.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/registry.example.com/privkey.pem;

  server_name registry.example.com;

  location /v2/ {
    auth_basic "Registry realm";
    auth_basic_user_file /etc/nginx/conf.d/nginx.htpasswd;

    # Do not allow connections from docker 1.5 and earlier
    # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents
    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
      return 404;
    }

    ## If $docker_distribution_api_version is empty, the header is not added.
    ## See the map directive above where this variable is defined.
    add_header 'Docker-Distribution-Api-Version' $docker_distribution_api_version always;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://127.0.0.1:5000;
    proxy_read_timeout 900;
  }

  include snippets/letsencrypt.conf;
}
```
