
# If a client requests for an unknown server name and there's no default server
# name defined, Nginx will serve the first server configuration found. To
# prevent this, we have to define a default server name and drop the request.
server {
    listen 80 default_server deferred;
    listen [::]:80 default_server deferred;
    server_name _;

    # Return 444 (No Response)
    return 444;
}