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