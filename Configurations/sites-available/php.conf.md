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

    # Files that will be used as index.
    index index.php index.html index.htm;

    location / {
        # First attempt to serve request as a file, then as a directory, then
        # pass it to index.php, then fallback to display 404 Not Found if
        # there's no index.php file.
        try_files $uri $uri/ /index.php$is_args$args =404;
    }

    # Pass PHP file to FastCGI server.
    location ~ \.php$ {
        include snippets/directive/fastcgi-php.conf;

        # With php-fpm or other unix sockets.
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;

        # With php-cgi or other tcp sockets).
        # fastcgi_pass 127.0.0.1:9000;
    }

    # Custom 404 page.
    error_page 404 /404.html;

    # Log configuration.
    error_log /var/lognginx/logs/malidkha.com_error.log error;
    access_log /var/lognginx/logs/malidkha.com_access.log main;

    # Include basic configuration.
    include snippets/basic.conf;
}