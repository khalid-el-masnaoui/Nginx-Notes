# Nginx Best Practices

## Overview
This document revisits the essential security and optimized-performance configurations for Nginx HTTP Server from a practical perspective.

By following the key recommendations outlined below, you can avoid common configuration errors and prevent security vulnerabilities as well as ensuring optimized performances


# Table of Contents

- **[Performance Optimizations](#performance-optimizations)**
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