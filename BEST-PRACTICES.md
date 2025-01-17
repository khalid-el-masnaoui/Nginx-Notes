# Nginx Best Practices

## Overview
This document revisits the essential security and optimized-performance configurations for Nginx HTTP Server from a practical perspective.

By following the key recommendations outlined below, you can avoid common configuration errors and prevent security vulnerabilities as well as ensuring optimized performances


# Table of Contents

- **[Performance Optimizations](#performance-optimizations)**
	- **[General Tuning](#general-tuning)**
	- **[Compression](#compression)**
	- **[Client Caching](#client-caching)**
	- **[SSL Caching](#ssl-caching)**
	- **[Other Configurations](#other-configurations)**
	- **[External System Settings](#external-system-settings)**

- **[Security](#securoty)**
	- **[Ensure Proper Permissions for Nginx Process Account](#ensure-proper-permissions-for-nginx-process-account)**
	- **[Ensure Nginx Version Information is Not Exposed](#ensure-nginx-version-information-is-not-exposed)**
	- **[Ensure the autoindex Feature is removed or disabled](#ensure-the-autoindex-feature-is-removed-or-disabled)**
	- **[Set Proper Permissions for Critical Directories and Files](#set-proper-permissions-for-critical-directories-and-files)**
	- **[Disabling Unused HTTP Methods](#disabling-unused-http-methods)**
	- **[Ensure Removal of Insecure and Outdated SSL Protocols and Ciphers](#ensure-removal-of-insecure-and-outdated-ssl-protocols-and-ciphers)**
	- **[Ensure Proper Permissions on SSL Certificate Files](#ensure-proper-permissions-on-ssl-certificate-files)**
	- **[Ensure Blocking of Arbitrary Host Connections Not Configured in the Web Server](#ensure-blocking-of-arbitrary-host-connections-not-configured-in-the-web-server)**
	- **[Limit Simultaneous Connections per IP](#limit-simultaneous-connections-per-ip)**
	- **[Limit Requests per IP](#10-limit-requests-per-ip)**
	- **[Inspect and Control Request Headers](#inspect-and-control-request-headers)**
	- **[Utilizing the Location Block](#utilizing-the-location-block)**
	- **[Restrict Access to the Proxy Servers and Local Networks](#restrict-access-to-the-proxy-servers-and-local-networks)**
	- **[FastCgi Timeouts](#fastcgi-timeouts)**
	- **[Hide Upstream Proxy Headers](#hide-upstream-proxy-headers)**
	- **[General Configurations Directives and Best Practices](#general-configurations-directives-and-best-practices)**

# Performance Optimizations
most of the configurations here , may need to be adjusted based on your needs and your traffic/server load, so please read and understand each configuration directive. 

## General Tuning

```nginx
# you must set worker processes based on your CPU cores, nginx does not benefit from setting more than that
worker_processes auto; #some last versions calculate it automatically

# number of file descriptors used for nginx
# the limit for the maximum FDs on the server is usually set by the OS.
# if you don't set FD's then OS settings will be used which is by default 2000
worker_rlimit_nofile 100000;

# only log critical errors
error_log /var/log/nginx/error.log crit;

# provides the configuration file context in which the directives that affect connection processing are specified.
events {
    # determines how much clients will be served per worker
    # max clients = worker_connections * worker_processes
    # Should be equal to `ulimit -n / worker_processes`
    # max clients is also limited by the number of socket connections available on the system (~64k)
    #a process(Worker_process) can have n number of threads(worker_connections).
    worker_connections 4000;

    # optimized to serve many clients with each thread, essential for linux
    use epoll;

    # accept as many connections as possible, may flood worker connections if set too low
    multi_accept on;
}

http {
    # cache informations about FDs, frequently accessed files
    # can boost performance
    open_file_cache max=200000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    # to boost I/O on HDD we can disable access logs
    access_log off;

    # copies data between one FD and other from within the kernel
    # faster than read() + write()
    sendfile on;

    # send headers in one piece, it is better than sending them one by one
    tcp_nopush on;

	# don't buffer data-sends (disable Nagle algorithm). Good for sending
	# frequent small bursts of data in real time.
    tcp_nodelay on;

    # allow the server to close connection on non responding client, this will       free up memory
    reset_timedout_connection on;


    # server will close connection after this time -- default 75
    keepalive_timeout 30;

    # number of requests client can make over keep-alive -- for testing environment
    keepalive_requests 100000;
}
```

## Compression
One of the first things that many people try to do is to enable the gzip compression module available with nginx. The intention here is that the objects which the server sends to requesting clients will be smaller, and thus faster to send.

However this involves the trade-off common to tuning, performing the compression takes CPU resources from your server, which frequently means that you'd be better off not enabling it at all.

Generally the best approach with compression is to only enable it for large files, and to avoid compressing things that are unlikely to be reduced in size (such as images, executables, and similar binary files).

With that in mind the following is a sensible configuration:
```nginx
gzip            on;
gzip_vary       on;
gzip_min_length 10240;
gzip_proxied    expired no-cache no-store private auth;
gzip_comp_level 4;
gzip_types
    # text/html is always compressed by HttpGzipModule
    text/css
    text/javascript
    text/xml
    text/plain
    text/x-component
    application/javascript
    application/x-javascript
    application/json
    application/xml
    application/rss+xml
    application/atom+xml
    font/truetype
    font/opentype
    application/vnd.ms-fontobject
    image/svg+xml;
gzip_disable "MSIE [1-6]\.";

```
This enables compression for files that are over 10k.

## Client Caching
A simple way of avoiding your server handling requests is if remote clients believe their content is already up-to-date.

To do this you want to set suitable cache-friendly headers, and a simple way of doing that is to declare that all images, etc, are fixed for a given period of time:
```nginx
# favicon.ico
location = /favicon.ico {
    log_not_found off;
    access_log    off;
}


# assets, media
location ~* \.(?:css(\.map)?|js(\.map)?|jpe?g|png|gif|ico|cur|heic|webp|tiff?|mp3|m4a|aac|ogg|midi?|wav|mp4|mov|webm|mpe?g|avi|ogv|flv|wmv)$ {
    expires    7d;
    add_header Cache-Control "public";
    access_log off;
    #add_header Vary: Accept-Encoding; enabled by gzip
}

# svg, fonts
location ~* \.(?:svgz?|ttf|ttc|otf|eot|woff2?)$ {
    add_header Access-Control-Allow-Origin "*";
    expires    7d;
    access_log off;
}


```

## SSL Caching

```nginx
ssl_session_timeout    1d;
ssl_session_cache      shared:SSL:128m;
ssl_session_tickets off;
ssl_buffer_size 1400;
```

## Other Configurations

####  Enable PCRE JIT to speed up processing of regular expressions

> Enables the use of JIT for regular expressions to speed-up their processing. Specifically, checking rules can be time-consuming, especially complex regular expression (regex) conditions.

```nginx
# In global context:
pcre_jit on;
```

#### Enable 0-RTT for TLSv1.3
> Generally speaking, 0-RTT is safe for most web sites and applications. If your web application does strange things and you’re concerned about its replay safety, consider not using 0-RTT until you can be certain that there are no negative effects. [...] TLS 1.3 is a big step forward for web performance and security. By combining TLS 1.3 with 0-RTT, the performance gains are even more dramatic.

> Including 0-RTT (Zero Round Trip Time Resumption) results in a significant increase in efficiency and connection times. TLS 1.3 has a faster handshake that completes in 1-RTT. Additionally, it has a particular session resumption mode where, under certain conditions, it is possible to send data to the server on the first flight (0-RTT).

```nginx
  ssl_protocols TLSv1.2 TLSv1.3;
  # To enable 0-RTT (TLS 1.3):
  ssl_early_data on;
```

**Important** : 0-RTT for TLSv1.3 creates a significant security risk called **Replay Attack** , check below for [security](#prevent-replay-attacks-on-0-rtt) best practices to mitigate/eliminate such risk.

#### Activate the cache for connections to upstream servers

> This can greatly reduce the number of new TCP connections, as NGINX can now reuse its existing connections (`keepalive`) per upstream.

> If your upstream server supports Keep-Alive in its config, NGINX will now reuse existing TCP connections without creating new ones. This can greatly reduce the number of sockets in `TIME_WAIT` TCP connections on a busy servers (less work for OS to establish new connections, less packets on a network).

> Keep-Alive connections are only supported as of HTTP/1.1.

```nginx
# Upstream context:
upstream backend {

  # Sets the maximum number of idle keepalive connections to upstream servers
  # that are preserved in the cache of each worker process.
  keepalive 16;

}

# Server/location contexts:
server {

  ...

  location / {

    # By default only talks HTTP/1 to the upstream,
    # keepalive is only enabled in HTTP/1.1:
    proxy_http_version 1.1;

    # Remove the Connection header if the client sends it,
    # it could be "close" to close a keepalive connection:
    proxy_set_header Connection "";

    ...

  }

}
```

## External System Settings

**Increase The Maximum Number Of Open Files (`nofile` limit) – Linux**

*_nofile_* :  value that refers to the maximum number of open files for a single process.

```bash
[root@localhost ~]# vim /etc/security/limits.conf
...
www-data soft nofile 200000 # <== add this line
www-data hard nofile 200000 # <== add this line
...
[root@localhost ~]# sysctl -p
[root@localhost ~]# vim /etc/pam.d/common-session
...
session required pam_limits.so # <== add this line
...
[root@localhost ~]# vim /etc/default/nginx
...
ULIMIT="-n 65535" # <== add this line
```

To confirm the limits per nginx process is taking effect

```bash
[root@localhost ~]# ps -ef | grep "nginx" #<== get one of nginx workers pid

cat /proc/{PID}/limits #<== displays the limit
```


**other linux tuning**

**Note** : We wont be covering linux tunning in details here, you can check my repository (**Linux-Server**) for that

Change the following kernel parameters to increase the number of ephemeral ports,  increase the allowed listen backlog and other tuned configurations.

```bash
[root@localhost ~]# vim /etc/sysctl.conf

#buffers
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# ipv4
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_max_syn_backlog=81920
net.ipv4.tcp_synack_retries=3
net.ipv4.tcp_syn_retries=3
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_tw_reuse = 1
#net.ipv4.tcp_tw_recycle = 1
net.ipv4.ip_local_port_range = 20000 65000
net.ipv4.tcp_max_tw_buckets = 200000
net.ipv4.route.max_size = 5242880
net.ipv4.tcp_fastopen = 3

...
[root@localhost ~]# sysctl -p
```


# Security

## Ensure Proper Permissions for Nginx Process Account
Nginx HTTP Server typically runs under the default `nginx`, `www-data` user account (or another non-privileged account). 

It is important to ensure that Nginx is not running under a privileged account, such as `root`. If it is, it must be changed to a non-privileged user to prevent potential security risks.


**Audit:**
- Verify the current user account under which Nginx is running, execute the following command:
  ```bash
  [root@localhost ~]# ps -ef | grep nginx
  root      626653       1  0 Apr08 ?        00:00:00 nginx: master process /usr/sbin/nginx
  nginx     626654  626653  0 Apr08 ?        00:00:00 nginx: worker process
  nginx     626655  626653  0 Apr08 ?        00:00:01 nginx: worker process
  nginx     626656  626653  0 Apr08 ?        00:00:00 nginx: worker process
  nginx     626657  626653  0 Apr08 ?        00:00:00 nginx: worker process
  ```

**Remediation:**
- If Nginx is running under the `root` account or any other account with elevated privileges, modify the configuration to run under a non-privileged account such as `nginx`, `nobody`, `daemon`:
  ```bash
  [root@localhost ~]# vim /etc/nginx/nginx.conf
  ...
  user nginx;   # <== # <== Set to a non-privileged account e.g. nginx, nobody, daemon
  ...
  ```

(1) Check sudo privileges:
- The Nginx process account must not have sudo privileges:
  ```bash
  [root@localhost ~]# sudo -l -U nginx
  User nginx is not allowed to run sudo on pg-sec-mqrelay01
  ```

(2) Verify shell access:
- The Nginx process account should not have access to a login shell:
  ```bash
  [root@localhost ~]# cat /etc/passwd | grep -i nginx
  nginx:x:498:498:Nginx web server:/var/lib/nginx:/sbin/nologin
  ```

(3) Ensure password is locked:
- The password for the Nginx process account should be locked:
  ```bash
  [root@localhost ~]# passwd -S nginx
  nginx LK 2023-10-16 -1 -1 -1 -1 (Password locked.)
  ```

## Ensure Nginx Version Information is Not Exposed
By default, Nginx HTTP Server displays the version information in the HTTP response header. This can provide attackers with insights into potential vulnerabilities. Hiding the version information mitigates this risk.

**Audit:**
- Verify that Nginx version information is exposed in the HTTP response header:
  ```nginx
  [root@localhost ~]# curl -i -k http://127.0.0.1
  HTTP/1.1 200
  Server: nginx/1.26.1
  ...
  ```
**Remediation:**
- To prevent the version information from being exposed, modify the following settings in the Nginx configuration:
  ```nginx
  [root@localhost ~]# vim /etc/nginx/nginx.conf
  http {
        ...
        server_tokens off;  # <== Set to hide version information
        ...
  ```

## Ensure the autoindex Feature is removed or disabled
The autoindex feature provides directory listing functionality. Directory listing can expose critical information about the web application and server, and should be disabled by default.

**Audit:**
- Verify whether directory listing is displayed or the autoindex feature is enabled.
  ```nginx
  [root@localhost ~]# vim /etc/nginx/nginx.conf
  ...
  
  location / {        
      root   /usr/local/nginx/html/html;
      autoindex on;
      index  index.html index.htm;
  }
  ```

**Remediation:**
- To disable the autoindex feature, remove or comment out the autoindex directive in the Nginx configuration:
  ```nginx
  [root@localhost ~]# vim /etc/nginx/nginx.conf
  ...

  location / {        
      root   /usr/local/nginx/html/html;
      autoindex on;                 # <== Remove or comment out
      index  index.html index.htm;  # <== Remove if necessary
  }
  ```

## Set Proper Permissions for Critical Directories and Files
Ensure that the Nginx installation directory and configuration files are accessible only to the "root" user. The permissions for these directories and files should be checked and modified if any other users have access rights.

**Audit:**
- Verify the permissions of Nginx's installation directory and configuration files.
  ```bash
  [root@localhost ~]# ls -al /etc/nginx/
  total 88
  drwxr-xr-x.  4 root  root  4096 Aug 12 11:28 .
  drwxr-xr-x. 87 root  root  8192 Aug  6 11:46 ..
  drwxr-xr-x.  4 root  root    68 Aug 12 11:06 conf.d
  drwxr-xr-x.  2 root  root     6 Jun 10  2021 default.d
  -rw-r--r--.  1 root  root  1077 Jun 10  2021 fastcgi.conf
  -rw-r--r--.  1 root  root  1077 Jun 10  2021 fastcgi.conf.default
  -rw-r--r--.  1 root  root  1007 Jun 10  2021 fastcgi_params
  -rw-r--r--.  1 root  root  1007 Jun 10  2021 fastcgi_params.default
  -rw-r--r--.  1 root  root  2837 Jun 10  2021 koi-utf
  -rw-r--r--.  1 root  root  2223 Jun 10  2021 koi-win
  -rw-r--r--.  1 root  root  5170 Jun 10  2021 mime.types
  -rw-r--r--.  1 root  root  5170 Jun 10  2021 mime.types.default
  -rw-r--r--.  1 root  root   927 Oct 16  2023 nginx.conf
  -rw-r--r--.  1 root  root  2656 Jun 10  2021 nginx.conf.default
  -rw-r--r--.  1 root  root   636 Jun 10  2021 scgi_params
  -rw-r--r--.  1 root  root   636 Jun 10  2021 scgi_params.default
  -rw-r--r--.  1 root  root   664 Jun 10  2021 uwsgi_params
  -rw-r--r--.  1 root  root   664 Jun 10  2021 uwsgi_params.default
  -rw-r--r--.  1 root  root  3610 Jun 10  2021 win-utf
  ```

**Remediation:**
- Ensure that all Nginx directories and configuration files are owned by root:root.
- Adjust the permissions so that others (non-owners) cannot access the directories and files.
  ```bash
  [root@localhost ~]# chown root:root -R /etc/nginx
  [root@localhost ~]# chmod o-rwx -R /etc/nginx
  ```

## Disabling Unused HTTP Methods
When unused HTTP methods are enabled on a web service, they can expose vulnerabilities or unintended behaviors that may lead to abuse and become an attack vector. 

It is important to disable any methods that are not in use, particularly the `TRACE` method, which is used for debugging and is generally unnecessary for normal web service operations.

**HTTP Methods:**

| Name    | Description                                                                                                                                         |
|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| OPTIONS | 	Used to check the list of supported methods by the web server.                                                                                     |
| HEAD    | 	The server sends only the header information in the response. It is often used to check the existence of a page by receiving the HTTP status code. |
| GET     | 	Similar to POST, but does not handle form input; typically used to retrieve information such as lists in forums via URI transmission.              |
| POST    | 	Used when sending data from a client to a web application for processing.                                                                          |
| PUT     | 	Similar to POST, used to save content specified by the client to the server.                                                                       |
| DELETE  | 	Used by the client to delete files on the server.                                                                                                  |
| TRACE   | 	Used to invoke a loopback message on the web server to trace the data transmission path.                                                           |
| CONNECT | 	Used to request proxy functionality from the web server.                                                                                           |

**Audit:**
- Verify that which HTTP methods are available on your web server:
  ```
  [root@localhost ~]# curl -i -k -X OPTIONS http://example.com
  HTTP/1.1 200 OK
  Server: nginx
  Date: Mon, 12 Aug 2024 03:00:29 GMT
  Content-Type: text/html; charset=utf-8
  Content-Length: 0
  Connection: keep-alive
  Allow: GET, HEAD, OPTIONS, PUT, DELETE, POST, PATCH
  ...
  ```

**Remediation:**
- Disable TRACE Method
- Enable Only Necessary HTTP Methods:
  ```nginx
  [root@localhost ~]# vim /etc/nginx/nginx.conf
  ...
  
  server {
      ...
      # Allow only specified methods (e.g., HEAD, GET, POST)
      if ($request_method !~ ^(HEAD|GET|POST)$ ){
              return 403;
        }
  }
  ...
  ```
***Note: The $request_method directive can be applied at the server or location block level as needed.***

## Ensure Removal of Insecure and Outdated SSL Protocols and Ciphers
When configuring SSL, outdated and insecure protocols or algorithms should be avoided. 

Major browsers like Chrome, Microsoft Edge, and Firefox have dropped support for TLS 1.0 and TLS 1.1 due to security vulnerabilities such as POODLE and BEAST. These older protocols are known to be vulnerable to various attacks.

| No. | CVE-ID        | Vulnerability Name                                      | Description                                                                |
|-----|---------------|---------------------------------------------------------|----------------------------------------------------------------------------|
| 1   | CVE-2014-3566 | POODLE (Padding Oracle On Downgraded Legacy Encryption) | Exploits outdated encryption methods, enabling protocol downgrade attacks. |
| 2   | CVE-2011-3389 | BEAST (Browser Exploit Against SSL/TLS)                 | Allows attackers to decrypt HTTPS cookies and hijack sessions.             |

**Deprecated Cipher Algorithms**
- The following cipher algorithms are considered insecure and should not be used:
  - RC4 
  - DES 
  - 3DES 
  - MD5 
  - aNULL 
  - eNULL 
  - EXPORT

**Audit:**
- Verify the SSL settings for `ssl_protocols` and `ssl_ciphers` configurations.
  ```nginx
  [root@localhost ~]# vim /etc/nginx/nginx.conf
  ...
  
  server {
          listen       443 ssl;
          server_name  example.com;  
          ...  
          ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 
          ssl_ciphers HIGH:MEDIUM:aNULL:MD5:EXPORT:RC4:eNULL:DES:3DES;
    }
  ```

**Remediation:**
- `ssl_protocols` should use TLS 1.2 or later.
- `ssl_ciphers` to utilize secure algorithms supported in TLS 1.2 or later.

**Recommended Configuration (General Use):**

- For general services (not bound by PCI-DSS), use the following configuration:
  ```
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;
  ```
- After configuration, verify the SSL security status using [SSL Labs' SSL Test.](https://www.ssllabs.com/ssltest/)
- Refer to [Mozilla's SSL Configuration Generator](https://ssl-config.mozilla.org/) for specific configurations based on your web server and OpenSSL version.


## Ensure Proper Permissions on SSL Certificate Files
In cases where a web service vulnerability allows remote access to web server permissions, attackers may attempt to access SSL certificate files, potentially leading to the exposure of the private key. If the SSL certificate's private key is leaked, an attacker could create a spoofed website that appears legitimate or use the key to distribute malware. This becomes especially dangerous when SSL certificate pinning is used, or if the certificate is a wildcard certificate, as it could necessitate replacing certificates across all servers and clients.

Ensure that SSL certificate files are secured by verifying and applying proper file permissions.

**Audit:**
- Verify the file ownership and permissions for the `ssl_certificate` and `ssl_certificate_key` files.
  ```nginx
  [root@localhost ~]# vim /etc/nginx/nginx.conf
  ...
  server {
          listen       443 ssl;
          server_name  example.com;
          ...
          ssl_certificate /etc/nginx/conf.d/cert/example.com_ssl.crt;
          ssl_certificate_key /etc/nginx/conf.d/cert/example.com_ssl.key;
          ...
    }
  
  
  [root@localhost ~]# ls -al /etc/nginx/conf.d/cert
  total 12
  drwxr-x---. 2 root root   64 Apr  8 14:47 .
  drwxr-x---. 4 root root   68 Aug 12 11:06 ..
  -rw-r--r--. 1 nobody root 7869 Apr  8 14:45 example.com_ssl.crt
  -rw-r--r--. 1 nobody root 1679 Apr  8 14:44 example.com_ssl.key
  ```

**Remediation:**
- Set the ownership of certificate and key files to `root:root`.
- Adjust the file permissions to `600` or `400` to prevent leaked.
  ```bash
  [root@localhost ~]# chown root:root -R /etc/nginx/conf.d/cert/example.com_ssl.crt
  [root@localhost ~]# chown root:root -R /etc/nginx/conf.d/cert/example.com_ssl.key
  [root@localhost ~]# chmod 400 /etc/nginx/conf.d/cert/example.com_ssl.crt
  [root@localhost ~]# chmod 400 /etc/nginx/conf.d/cert/example.com_ssl.key
  ```

## Ensure Blocking of Arbitrary Host Connections Not Configured in the Web Server
In a typical web server configuration, if someone knows the server's IP address, they might still be able to access the web service without knowing the valid hostname (domain). By isolating and blocking access to requests made to non-service hosts, you can reduce unnecessary connections that shouldn't lead to your service, thereby reducing server load and improving performance.

Blocking such arbitrary connections also enhances security by preventing various types of web service attacks, such as host header injection and IP address-based scanning.

Ensure that your web server is configured to only respond to requests made to the correct service host, while blocking all others.

**Audit:**
- Verify that unknown host requests are accepting when accessing the web server.

**(Vulnerable) Web server accepting connections with unknown host requests:**
  ```bash
  [root@localhost ~]# curl -k -v https://192.168.130.23 -H 'Host: invalid.host.com'
  *   Trying 192.168.130.23:443...
  * Connected to 192.168.130.23 (192.168.130.23) port 443
  * ALPN: curl offers h2,http/1.1
  * TLSv1.3 (OUT), TLS handshake, Client hello (1):
  * TLSv1.3 (IN), TLS handshake, Server hello (2):
  * TLSv1.2 (IN), TLS handshake, Certificate (11):
  * TLSv1.2 (IN), TLS handshake, Server key exchange (12):
  * TLSv1.2 (IN), TLS handshake, Server finished (14):
  * TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
  * TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
  * TLSv1.2 (OUT), TLS handshake, Finished (20):
  * TLSv1.2 (IN), TLS handshake, Finished (20):
  * SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384 / prime256v1 / rsaEncryption
  * ALPN: server did not agree on a protocol. Uses default.
  * Server certificate:
  *  subject: CN=*.example.com
  *  start date: Jan  2 00:00:00 2024 GMT
  *  expire date: Jan 20 23:59:59 2025 GMT
  *  issuer: C=GB; ST=Greater Manchester; L=Salford; O=Sectigo Limited; CN=Sectigo RSA Domain Validation Secure Server CA
  *  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
  *   Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
  *   Certificate level 1: Public key type RSA (2048/112 Bits/secBits), signed using sha384WithRSAEncryption
  *   Certificate level 2: Public key type RSA (4096/152 Bits/secBits), signed using sha384WithRSAEncryption
  * using HTTP/1.x
  > GET / HTTP/1.1
  > Host: invalid.host.com
  > User-Agent: curl/8.5.0
  > Accept: */*
  >
  < HTTP/1.1 200 OK
  < Date: Mon, 12 Aug 2024 05:35:40 GMT
  < Server: Nginx
  < Cache-Control: no-cache, no-store, must-revalidate
  < Expires: Thu, 01 Jan 1970 09:00:00 KST
  ...
  ```

**(Not Vulnerable) Web server not accepting connections with unknown host requests:**
  ```bash
  [root@localhost ~]# curl -k -v https://192.168.130.236 -H 'Host: invalid.host.com'
  *   Trying 192.168.130.236:443...
  * Connected to 192.168.130.236 (192.168.130.236) port 443 (#0)
  * ALPN: offers h2
  * ALPN: offers http/1.1
  * (304) (OUT), TLS handshake, Client hello (1):
  * (304) (IN), TLS handshake, Server hello (2):
  * (304) (IN), TLS handshake, Unknown (8):
  * (304) (IN), TLS handshake, Certificate (11):
  * (304) (IN), TLS handshake, CERT verify (15):
  * (304) (IN), TLS handshake, Finished (20):
  * (304) (OUT), TLS handshake, Finished (20):
  * SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
  * ALPN: server accepted http/1.1
  * Server certificate:
  *  subject: CN=*.example.com
  *  start date: Jan  2 00:00:00 2024 GMT
  *  expire date: Jan 20 23:59:59 2025 GMT
  *  issuer: C=GB; ST=Greater Manchester; L=Salford; O=Sectigo Limited; CN=Sectigo RSA Domain Validation Secure Server CA
  *  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
  *   Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
  *   Certificate level 1: Public key type RSA (2048/112 Bits/secBits), signed using sha384WithRSAEncryption
  *   Certificate level 2: Public key type RSA (4096/152 Bits/secBits), signed using sha384WithRSAEncryption
  * using HTTP/1.x
  > GET / HTTP/1.1
  > Host: invalid.host.com
  > User-Agent: curl/7.84.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 403 Forbidden
  < Server: Nginx
  < Date: Mon, 12 Aug 2024 05:32:49 GMT
  < Content-Type: text/html
  < Content-Length: 162
  < Connection: keep-alive
  <
  <html>
  <head><title>403 Forbidden</title></head>
  <body bgcolor="white">
  <center><h1>403 Forbidden</h1></center>
  <hr><center>Nginx</center>
  </body>
  </html>
  * Connection #0 to host 192.168.130.236 left intact
  ```

**Remediation:**
- Create a virtual host that processes and blocks requests with host headers that do not match the intended service hosts.
  ```nginx
  [root@localhost ~]# vim /etc/nginx/nginx.conf
  ...
  
  # VirtualHost to handle and block requests with unmatched host headers
  server {
          listen       80 default_server;
          listen       443 default_server ssl;
  
          error_log    /var/log/nginx/http.unknown-header.error.log;
          access_log   /var/log/nginx/http.unknown-header.access.log  main;
  
          ssl_certificate /etc/nginx/conf.d/cert/example.com_ssl.crt;
          ssl_certificate_key /etc/nginx/conf.d/cert/example.com_ssl.key;
          ssl_session_cache shared:SSL:1m;
          ssl_session_timeout  10m;
  
          ssl_protocols TLSv1.2 TLSv1.3;
          ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;
          
          location / {
               deny all;
          }
  }
  
  # VirtualHost for the valid service hostname (example.com)
  server {
          listen       80;
          server_name  example.com;
  
          location / {
            return 301 https://example.com$request_uri;
          }
          ...
  }
      
  server {
          listen       443 ssl;
          server_name  example.com;
  
          error_log    /var/log/nginx/example.com.error.log;
          access_log   /var/log/nginx/example.access.log  main;
  
          ssl_certificate /etc/nginx/conf.d/cert/example.com_ssl.crt;
          ssl_certificate_key /etc/nginx/conf.d/cert/example.com_ssl.key;
          ssl_session_cache shared:SSL:1m;
          ssl_session_timeout  10m;
  
          ssl_protocols TLSv1.2 TLSv1.3;
          ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;
          ...
  }
  ```

**Notes:**
- Ensuring that requests with incorrect host header are blocked is a critical part of web server security.
- `Many web servers are not configured this way by default`, making this an essential configuration for system administrators to apply.


  
## Limit Simultaneous Connections per IP
Limiting the number of simultaneous connections from a single IP address is a critical step in preventing resource exhaustion and ensuring the availability of services. By implementing connection limits, you can mitigate the risk of slow denial-of-service (DoS) attacks and maintain stable web server performance.

**Audit:**
- Verifying Simultaneous Connections
- To audit the current simultaneous connections per IP, you can monitor server logs or analyze connection counts using tools like Nginx’s built-in logging or external monitoring systems.

**Remediation:** 
- Configuring Connection Limits

(1) Define a Shared Memory Zone
- Use the limit_conn_zone directive in the http block to create a shared memory zone that tracks connections by IP. Example:
  ```nginx
  http {
      limit_conn_zone $binary_remote_addr zone=limitperip:10m;
      ...
  }
  ```
- `Memory Allocation:` 10MB can store approximately **160,000 unique IPs**.
- `Tracking Mechanism:` $binary_remote_addr tracks client IPs efficiently.

(2) Apply Connection Limits
- Use the limit_conn directive to enforce restrictions at the server or location level. Examples:
  ```nginx
  server {
      listen 443 ssl;
      server_name auth.example.com;
  
      # Allow a maximum of 10 simultaneous connections per IP
      limit_conn limitperip 10;
  }
  
  server {
      listen 443 ssl;
      server_name download.example.com;
  
      location /download/ {
          # Restrict simultaneous connections to 3 per IP for this location
          limit_conn limitperip 3;
          limit_conn_status 429;  # Return HTTP 429 (Too Many Requests)
      }
  }
  ```

**Notes:**
- The `ngx_http_limit_conn_module` module must be enabled.
- Fine-tune memory allocation and limits based on your server’s capacity and traffic profile.
- The configuration ensures that excessive connections from a single IP do not degrade server performance or accessibility for other users.

By enforcing simultaneous connection limits, you can protect your server from potential abuse and maintain service quality under high-load scenarios.

## Limit Requests per IP
Rate limiting per IP address in Nginx helps maintain stable web services by controlling the number of requests a client can make within a given timeframe. This configuration prevents abuse by users generating excessive requests and protects the server from being overwhelmed.

**Audit:**
- Verifying Request Limits
- To check if rate-limiting is configured:
  - Review the Nginx configuration file for `limit_req_zone` and `limit_req` directives.
  - Analyze the server logs to identify patterns of excessive requests or potential abuse.


**Remediation:**
- Configuring Rate Limits

(1) Define a Shared Memory Zone
- Use the `limit_req_zone` directive to allocate a shared memory zone that tracks request rates by IP. Example:
  ```nginx
  http {
      limit_req_zone $binary_remote_addr zone=ratelimit:10m rate=5r/s;
      ...
  }
  ```
- `Memory Allocation`: 10MB can store approximately 160,000 unique IPs.
- `Rate:` Limits each IP to 5 requests per second (`5r/s`).

(2) Apply Request Limits
- Use the `limit_req` directive in the `location` block to enforce the rate limits. Example:
  ```nginx
  server {
      listen 443 ssl;
      server_name auth.example.com;
  
      location / {
          limit_req zone=ratelimit burst=10 nodelay;
          limit_req_status 429;  # Return HTTP 429 (Too Many Requests)
      }
  }
  ```
- `Burst:` Allows up to 10 requests to be queued during traffic spikes.
- `nodelay:` Ensures all requests in the burst are processed immediately without delay.

**Notes:**
- The `ngx_http_limit_conn_module` module must be enabled.
- Adjust the rate and burst settings to align with your application's traffic profile.
- Rate limits help maintain consistent performance by preventing abuse while accommodating occasional bursts of traffic.

By implementing IP-based rate limits, you can protect your server from misuse and ensure a reliable experience for legitimate users.


## Inspect and Control Request Headers
Web servers can inspect and control HTTP request headers to determine the presence or value of specific headers. This capability enables conditional access, such as blocking specific clients or permitting predefined user agents (e.g., mobile apps). It is commonly applied for:

- Blocking known attack tools or abusive scanning software.
- Allowing access only to predefined clients or bots.
- Securing APIs by validating authorization headers.

**Audit:**
- Verifying Header Restrictions is applied
  - Review the Nginx configuration for if directives that check headers like User-Agent or Authorization.
  - Check server logs for unauthorized requests and verify appropriate error responses are sent.


**Remediation:**
- Configuring Header Restrictions

(1) Block Known Vulnerability Scanners
- Example configuration to block common web vulnerability scanning tools:
  ```nginx
  server {
      listen 443 ssl;
      server_name download.example.com;
  
      # Block requests with known attack tool User-Agent strings
      if ($http_user_agent ~* "Paros|ZmEu|nikto|dirbuster|sqlmap|openvas|w3af|Morfeus|Zollard|Arachni|Brutus|bsqlbf|Grendel-Scan|Havij|Hydra|N-Stealth|Netsparker|Pangolin|pmafind|webinspect") {
          return 404;  # Return HTTP 404 Not Found
      }
  }
  ```

(2) Restrict Specific Headers for Certain URLs
- Example to enforce valid authorization headers for API access:
  ```nginx
  server {
      listen 443 ssl;
      server_name download.example.com;
  
      location /api {
          proxy_http_version 1.1;
  
          # Only allow requests with Authorization header starting with "Bearer"
          if ($http_authorization !~ "^Bearer") {
              return 401;  # Return HTTP 401 Unauthorized
          }
  
          proxy_pass http://app:3000/;
      }
  }
  ```

(3) Allow Only Specific Clients
- Example to permit access to predefined User-Agent strings:
  ```nginx
  http {
      if ($http_user_agent !~ "Agent.MyApp") {
          return 404;  # Return HTTP 404 Not Found
      }
  }
  
  server {
      listen 443 ssl;
      server_name auth.example.com;
  }
  
  server {
      listen 443 ssl;
      server_name download.example.com;
  }
  ```
  
**Notes:**
- The `$http_` variables in Nginx provide access to HTTP headers.
- Regular expressions (`~` or `~*`) are used to match patterns in headers.
- Use conditional blocks (`if`) judiciously to avoid unintended behavior or performance issues.
- Return appropriate HTTP status codes (e.g., `404`, `401`) to signal blocked or unauthorized access.

## Utilizing the Location Block
The location block in Nginx is used to define how specific URIs are handled. Each virtual host can use multiple location blocks, which can be configured with different priorities. This allows fine-grained control over request handling.

**Basic Usage**
- The syntax for a `location` block is as follows:
  ```nginx
  location [modifier] [URI] {
      ...
  }
  ```

**Matching Priorities**

| Priority | Modifier | Description                                | Example                                              |
|----------|----------|--------------------------------------------|------------------------------------------------------|
| 1        | `=`      | Matches exact URIs.                        | `location = /favicon.ico { # do something }`         |
| 2        | `^~`     | Matches URIs with a preferred prefix.      | `location ^~ /images/ { # do something }`            |
| 3        | `~`      | Matches URIs using case-sensitive regex.   | `location ~ \.php$ { # do something }`               |
| 4        | `~*`     | Matches URIs using case-insensitive regex. | `location ~* \.(jpg\|jpeg\|png)$ { # do something }` |                         |                                          |
| 5        | `/`      | Matches all other URIs (fallback).         | `location /docs/ { # do something }`                 |

**Remediation:**
- Example Use Cases of the location Block

(1) Serve Specific File for Dynamic URLs
- Serve `/cafe-detail/index.html` for requests like `/cafes/12345`
  ```nginx
  location ~ ^/cafes/[1-9]+$ {
      try_files $uri /cafes/cafe-detail/index.html;
  }
  ```

(2) Block Access to Hidden(.Dot) Files
- Deny access to hidden files (starting with a dot):
  ```nginx
  location ~ /\. {
      deny all;
      return 404;
  }
  ```

(3) Serve Static Files with Caching
- Serve `css` and `js` files with a 30-day cache:
  ```nginx
  location ~* \.(css|js)$ {
      root /var/www/static;
      expires 30d;
  }
  ```

(4) Restrict Access to Sensitive Files
- Block access to `.sql` and `.bak` files:
  ```nginx
  location ~* \.(sql|bak)$ {
      deny all;
      return 404;
  }
  ```

(5) Add Content-Type for API Responses
- Add a Content-Type header for `.json` or `.xml` responses:
  ```nginx
  location /api/ {
      location ~* \.(json|xml)$ {
          add_header Content-Type application/json;
      }
  }
  ```

(6) Redirect Mobile Users
- Redirect mobile users to a mobile-specific site version:
  ```nginx
  location / {
      set $mobile_rewrite do_not_perform;
      if ($http_user_agent ~* "(android|bb\d+|meego).+mobile|avantgo|bada\/|blackberry|blazer|compal|elaine|fennec|...)") {
          set $mobile_rewrite perform;
      }
      if ($mobile_rewrite = perform) {
          rewrite ^ /mobile$uri redirect;
      }
  }
  ```

(7) Enable File Downloads
- Set response headers for file downloads:
  ```nginx
  location ~* ^/download/.*\.(zip|tar\.gz)$ {
      root /var/www/files;
      add_header Content-Disposition "attachment";
  }
  ```

(8) Rate Limiting for Login Attempts
- Limit request rates for WordPress login pages:
  ```nginx
  location = /wp-login.php {
      limit_req zone=one burst=1 nodelay;
      include fastcgi_params;
      fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
  }
  location ~* /wp-admin/.*\.php$ {
      auth_basic "Restricted Access";
      auth_basic_user_file /etc/nginx/.htpasswd;
      include fastcgi_params;
      fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
  }
  ```

(9) Rewrite API Versions
- Process API requests with version-specific rewrites:
  ```nginx
  location /api/ {
      rewrite ^/api/v1/(.*)$ /api.php?version=1&request=$1 break;
      rewrite ^/api/v2/(.*)$ /api.php?version=2&request=$1 break;
  }
  ```

(10) Redirect Old Blog URLs
- Rewrite legacy blog URLs to a new format:
  ```nginx
  location /blog/ {
      rewrite ^/blog/(\d{4})/(\d{2})/(.*)$ /articles/$1-$2-$3 permanent;
  }
  ```

(11) Disable Caching for Sensitive Pages
- Prevent caching for certain pages:
  ```nginx
  location ~ ^/payments/v[1-2]/daily/download {
      add_header Cache-Control 'no-cache, no-store, must-revalidate, proxy-revalidate, max-age=0';
      add_header Pragma no-cache;
      expires 0;
  }
  ```
  
Using the location block effectively enhances web server performance, security, and user experience.


## Restrict Access to the Proxy Servers and Local Networks
This section contains a complete configuration examples for CloudFlare-only nginx, The goal is to close your web server to cloudflare-only traffic (as well as LANs) using firewall rules and [CloudFlare IP list](https://www.cloudflare.com/ips/).

**Note** : We wont be covering the firewall rules for this , you can check my repository (**Linux-Server**) for that

  ```nginx
  [root@localhost ~]# vim /etc/nginx/nginx.conf
  ...
  
  http {
      ...
      # Cloudlfare ips
      include nginxconfig.io/cloudflare_real_ips.conf;
      include nginxconfig.io/cloudflare_whitelist.conf;
  }
  ...
  ```



  ```nginx
  [root@localhost ~]# vim /etc/nginx/nginxconfig.io/cloudflare_whitelist.conf
  ...
  
geo $realip_remote_addr $cloudflare_ip {
  default 0;
  127.0.0.0/8 1;
  192.168.1.0/24 1;
  173.245.48.0/20 1;
  103.21.244.0/22 1;
  103.22.200.0/22 1;
  103.31.4.0/22 1;
  141.101.64.0/18 1;
  108.162.192.0/18 1;
  190.93.240.0/20 1;
  188.114.96.0/20 1;
  197.234.240.0/22 1;
  198.41.128.0/17 1;
  162.158.0.0/15 1;
  104.16.0.0/13 1;
  104.24.0.0/14 1;
  172.64.0.0/13 1;
  131.0.72.0/22 1;
}

# if you wish to only allow cloudflare IP's add this to your site block for each host:
#if ( != 1) {
#       return 403;
#}

  ...
  ```

  ```nginx
  server {
    ...
    # Cloudlfare ips
	if ($cloudflare_ip != 1) {
		return 403;
	}
	...
  }
  ```

To ensure the cloudflare ips are automatically included , we prepare a bash script to ensure that CloudFlare ip addresses are automatically refreshed every week (you can choose any frequency), and nginx will be realoded when synchronization is completed. 

```bash
[root@localhost ~]# crontab -e
...
#IPS
@weekly /bin/bash /etc/nginx/scripts/cloudflare-ips.sh
...
  ```

**Note** : the script below add cloudflare-ips, internal ips and loopack ips , and allow connection from these ips using "ufw firewall rules", if you dont need this you can set `UFW_RULES=false`

```bash
[root@localhost ~]# vim /etc/nginx/scripts/cloudflare-ips.sh
...
#!/bin/bash

CLOUDFLARE_REAL_IPS_PATH=/etc/nginx/nginxconfig.io/cloudflare_real_ips.conf
CLOUDFLARE_WHITELIST_PATH=/etc/nginx/nginxconfig.io/cloudflare_whitelist.conf

UFW_RULES=true
TRUSTED_IP_LOOPACK="127.0.0.0/8"
TRUSTED_IP_INTERNAL="192.168.1.0/24"

set -e

for file in $CLOUDFLARE_REAL_IPS_PATH $CLOUDFLARE_WHITELIST_PATH; do
        echo "# https://www.cloudflare.com/ips" > $file
        echo "# Generated at $(LC_ALL=C date)" >> $file
done

echo "geo \$realip_remote_addr \$cloudflare_ip {
        default 0;
    $TRUSTED_IP_LOOPACK 1;
    $TRUSTED_IP_INTERNAL 1;" >> $CLOUDFLARE_WHITELIST_PATH

for type in v4; do
        echo "# IP$type"

        for ip in `curl https://www.cloudflare.com/ips-$type`; do
                echo "set_real_ip_from $ip;" >> $CLOUDFLARE_REAL_IPS_PATH;
                echo "  $ip 1;" >> $CLOUDFLARE_WHITELIST_PATH;
                if [ $UFW_RULES = true ] ; then
                        ufw allow from $ip to any app 'Nginx Full'  comment "cloudflare"
                        ufw allow from $ip to any app 'Nginx Full'  comment "cloudflare"
                fi
        done
done

echo "}
# if you wish to only allow cloudflare IP's add this to your site block for each host:
#if ($cloudflare_ip != 1) {
#       return 403;
#}" >> $CLOUDFLARE_WHITELIST_PATH
echo "real_ip_header CF-Connecting-IP;" >> $CLOUDFLARE_REAL_IPS_PATH

nginx -t && systemctl reload nginx # test configuration and reload nginx

# reload firewall
if [ $UFW_RULES = true ] ; then
        ufw allow from $TRUSTED_IP_LOOPACK to any app 'Nginx Full' comment "internal-subnet"
    ufw allow from $TRUSTED_IP_INTERNAL to any app 'Nginx Full' comment "internal-subnet"
        /usr/sbin/ufw reload > /dev/null
fi

  ```


## FastCgi Timeouts

```nginx
location ~ \.php$ {
fastcgi_buffer_size 128k;  
fastcgi_buffers 256 16k;  
fastcgi_busy_buffers_size 256k;  
fastcgi_temp_file_write_size 256k;  
fastcgi_read_timeout 600;#if need it*
fastcgi_send_timeout 600;#if need it*
fastcgi_connect_timeout 600;#if need it*
...
```

##  Hide upstream proxy headers


> When NGINX is used to proxy requests to an upstream server (such as a PHP-FPM instance), it can be beneficial to hide certain headers sent in the upstream response (e.g. the version of PHP running).

> You should use `proxy_hide_header` (or Lua module) to hide/remove headers from upstream servers returned to your NGINX reverse proxy (and consequently to the client).

###### Example

```nginx
# Hide some standard response headers:
proxy_hide_header X-Powered-By;
proxy_hide_header X-AspNetMvc-Version;
proxy_hide_header X-AspNet-Version;
proxy_hide_header X-Drupal-Cache;

# Hide some Amazon S3 specific response headers:
proxy_hide_header X-Amz-Id-2;
proxy_hide_header X-Amz-Request-Id;

# Hide other risky response headers:
proxy_hide_header X-Runtime;
```

## Prevent Replay Attacks on Zero Round-Trip Time
> 0-RTT Handshakes is part of the replacement of TLS Session Resumption and was inspired by the QUIC Protocol.

> 0-RTT creates a significant security risk. With 0-RTT, a threat actor can intercept an encrypted client message and resend it to the server, tricking the server into improperly extending trust to the threat actor and thus potentially granting the threat actor access to sensitive data.

> To protect against such attacks at the application layer, the `$ssl_early_data` variable should be used. You'll also need to ensure that the `Early-Data` header is passed to your application. `$ssl_early_data` returns 1 if TLS 1.3 early data is used and the handshake is not complete.

> However, as part of the upgrade, you should disable 0-RTT until you can audit your application for this class of vulnerability.

```nginx
  location / {

    proxy_pass http://backend_x20;
    # It protect against such attacks at the application layer:
    proxy_set_header Early-Data $ssl_early_data;

  }
```

You need this sort of check on everything you want Replay protection on (eg. `POST /transfer_money`).

While you can leave it off of something that has no side effects (eg. `GET /account_balance`).

Because the attacker cannot decode the payload in the replay, the GET has no teeth and you can allow those requests to use TLS Early Data.

**example** : Using it with PHP-FPM by setting a variable

```nginx
fastcgi_param  TLS_EARLY_DATA     $ssl_early_data;
```

Then in PHP you need to perform a check for any request that is at risk for a replay attack:

```php
if ($_SERVER['TLS_EARLY_DATA'] === '1') {
    http_response_code(425);
    exit;
}
```

## General Configurations Directives and Best Practices


  ```nginx
# security headers
add_header Set-Cookie                "Path=/; HttpOnly; Secure";


# This header enables the Cross-site scripting (XSS) filter built into most recent web browsers.
# It's usually enabled by default anyway, so the role of this header is to re-enable the filter for
# this particular website if it was disabled by the user.
add_header X-XSS-Protection          "1; mode=block" always;


# when serving user-supplied content, include a X-Content-Type-Options: nosniff header along with the Content-Type: header,
# to disable content-type sniffing on some browsers.
add_header X-Content-Type-Options    "nosniff" always;


add_header Referrer-Policy           "no-referrer-when-downgrade" always;
add_header Permissions-Policy        "interest-cohort=()" always;



# config to enable HSTS(HTTP Strict Transport Security) https://developer.mozilla.org/en-US/docs/Security/HTTP_Strict_Transport_Security|
# to avoid ssl stripping
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;



# config to don't allow the browser to render the page inside an frame or iframe
# and avoid clickjacking http://en.wikipedia.org/wiki/Clickjacking|
# if you need to allow [i]frames, you can use SAMEORIGIN or even set an uri with ALLOW-FROM uri
add_header X-Frame-Options           "DENY" always;


# with Content Security Policy (CSP) enabled(and a browser that supports it),you can tell the browser that it can only download content from the domains you explicitly allow
# I need to change our application code so we can increase security by disabling 'unsafe-inline' 'unsafe-eval'|
# directives for css and js(if you have inline css or js, you will need to keep it too).|
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://ssl.google-analytics.com https://assets.zendesk.com https://connect.facebook.net; img-src 'self' https://ssl.google-analytics.com https://s-static.ak.facebook.com https://assets.zendesk.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://assets.zendesk.com; font-src 'self' https://themes.googleusercontent.com; frame-src https://assets.zendesk.com https://www.facebook.com https://s-static.ak.facebook.com https://tautt.zendesk.com; object-src 'none'";


# . files
location ~ /\.(?!well-known) {
    deny all;
}

location /LICENSE {
    deny all;
}

location ~ ^\. {
    deny all;
}

location ~ \.(ini|log|sh|md|txt|json|lock)$ {
    deny all;
}

location ~ /\.git {
    deny all;
}

# Block common hacks

location ~* .(display_errors|set_time_limit|allow_url_include.*disable_functions.*open_basedir|set_magic_quotes_runtime|webconfig.txt.php|file_put_contentssever_root|wlwmanifest) {
    deny all;
}

location ~* .(globals|encode|localhost|loopback|xmlrpc|revslider|roundcube|webdav|smtp|http\:|soap|w00tw00t) {
    deny all;
}

# Protect other sensitive files

location ~* \.(engine|inc|info|install|make|module|profile|test|po|sh|.*sql|theme|tpl(\.php)?|xtmpl)$|^(\..*|Entries.*|Repository|Root|Tag|Template)$|\.php_ {
    deny all;
}

# Help guard against SQL injection

location ~* .(\;|\'|\"|%22).*(request|insert|union|declare|drop)$ {
    deny all;
}


# Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
ssl_dhparam /etc/nginx/ssl/dhparam.pem;

# enables server-side protection from BEAST attacks
ssl_prefer_server_ciphers on;

# disable SSLv3(enabled by default since nginx 0.8.19) since it's less secure 
ssl_protocols TLSv1.2 TLSv1.3;

# ciphers chosen for forward secrecy and compatibility
ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';

# enable ocsp stapling (mechanism by which a site can convey certificate revocation information to visitors in a privacy-preserving, scalable manner)
resolver 1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4 208.67.222.222 208.67.220.220 valid=1d;
resolver_timeout       2s;
ssl_stapling on;
ssl_stapling_verify on;


# if the request body size is more than the buffer size, then the entire (or partial)
# request body is written into a temporary file
client_body_buffer_size  128k;

# buffer size for reading client request header
client_header_buffer_size 3m;

# maximum number and size of buffers for large headers to read from client request
large_client_header_buffers 4 256k;

# read timeout for the request body from client
client_body_timeout   5;

# how long to wait for the client to send a request header (between packets
client_header_timeout 5;

# if client stop responding (between packets), free up memory-- default 60
send_timeout 2;

# allow the server to close connection on non responding client, this will free up memory
reset_timedout_connection on;

 ## Restricted Access directory by password in the access_list file.
  location ^~ /secure/ {
		allow 127.0.0.1/32;
		allow 10.10.10.0/24;
		deny all;
		auth_basic "RESTRICTED ACCESS";
		auth_basic_user_file /var/www/htdocs/secure/access_list;
	}
  ```


For even more details and further learning, you can check this github repository:
[Nginx Admin's Handbook](https://github.com/trimstray/nginx-admins-handbook)