# Nginx Architecture

Notes about nginx , nginx architecture, what is and how FastCGI works and other nginx related concepts.


## Introduction

##### What is Nginx?

**_NGINX_** was originally designed as a _web server_ with _high performance_, _reliability_ and low memory usage,  besides being a web server, Nginx also enables _reverse proxying_, _caching_, _load balancing_, and _media streaming_., started as  an attempt to answer the [C10k problem](https://en.wikipedia.org/wiki/C10k_problem). C10k is the challenge of managing ten thousand connections at the same time.

Today, there are even more connections that web servers have to handle. For that reason, NGINX offers an **_event-driven_** and **_asynchronous architecture**_. This feature makes NGINX one of the most reliable servers for speed and scalability.