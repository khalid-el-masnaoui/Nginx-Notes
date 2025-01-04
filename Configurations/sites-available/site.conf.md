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
    root /var/www/html/malidkha.com/public;

    # The charset.
    charset utf-8;

    # Files that will be used as index.
    index index.html index.htm;

    location / {
        # First attempt to serve request as a file, then as a directory, then
        # fallback to display 404 Not Found.
        try_files $uri $uri/ =404;
    }

    # Custom 404 page.
    error_page 404 /404.html;

    # Log configuration.
    error_log /var/log/nginx/logs/malidkha.com_error.log error;
    access_log /var/log/nginx/logs/malidkha.com_access.log main;

    # Include basic configuration.
    include snippets/basic.conf;
}