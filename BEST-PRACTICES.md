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