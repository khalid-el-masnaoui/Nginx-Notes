# Group of servers that will be proxied to.
upstream backend {
    server localhost:3000;
}

server {
    listen 80;
    listen [::]:80;

    # The www host server name.
    server_name www.malidkha.com;

    # Redirect to the non-www version.
    return 301 $scheme://malidkha.com$request_uri;
}

server {
    listen 80;
    listen [::]:80;

    # The non-www host server name.
    server_name malidkha.com;

    # The document root path.
    root /var/www/malidkha.com/public;

    # The charset.
    charset utf-8;

    location / {
        # First attempt to serve request as a file, then proxy it to the
        # backend group.
        try_files $uri @backend;
    }

    location @backend {
        proxy_pass http://backend;
        include snippets/directive/proxy.conf;
        include snippets/directive/websocket-proxy.conf;
    }

    # Log configuration.
    error_log /var/log/nginx/logs/malidkha.com_error.log error;
    access_log /var/log/nginx/logs/malidkha.com_access.log main;

    # Include basic configuration.
    include snippets/basic.conf;
}