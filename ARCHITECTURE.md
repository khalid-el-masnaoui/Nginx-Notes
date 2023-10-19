# Nginx Architecture

Notes about nginx , nginx architecture, what is and how FastCGI works and other nginx related concepts.


## Introduction

#### What is Nginx?

**_NGINX_** was originally designed as a _web server_ with _high performance_, _reliability_ and low memory usage,  besides being a web server, Nginx also enables _reverse proxying_, _caching_, _load balancing_, and _media streaming_., started as  an attempt to answer the [C10k problem](https://en.wikipedia.org/wiki/C10k_problem). C10k is the challenge of managing ten thousand connections at the same time.

Today, there are even more connections that web servers have to handle. For that reason, NGINX offers a **_single threaded_**, **_event-driven_** and **_asynchronous architecture_**. This feature makes NGINX one of the most reliable servers for speed and scalability.

#### Event-Driven Architecture

###### What is Event-driver Architecture?

Event-driven architecture is a _software architecture_ and model for application design. With an event-driven system, the capture, communication, processing, and persistence of events are the core structure of the solution. This differs from a traditional _request-driven_ model.

Event-driven architecture is defined as a software design pattern in which decoupled applications can **_asynchronously_** publish and subscribe to events via an _event broker_.

By introducing a middleman known as an event broker, event-driven architecture enables what’s called loose coupling of applications, meaning applications and devices don’t need to know where they are sending information, or where information they’re consuming comes from.

**Event-driven architecture** (**EDA**) is a software design pattern that enables an organization to detect “_events_” and act on them in real time or near real time. This pattern replaces the traditional “_request/response_” architecture where services would have to wait for a reply before they could move onto the next task. The flow of event-driven architecture is run by events and it is designed to respond to them or carry out some action in response to an event.

Event-driven architecture is often referred to as “**_asynchronous_**” communication. This means that the sender and recipient don’t have to wait for each other to move onto their next task. Systems are not dependent on that one message.

<p align="center">
<img src="./images/eda.svg"/>
</p>

###### What is an event?

An event is any significant occurrence or change in state for system hardware or software , or more broadly anything that can be noticed and recorded by an application or device, and shared with other applications and devices. All of the things that happen _within_ and _to_ your enterprise are events – customer requests, inventory updates, sensor readings, etc.

###### How does event-driven architecture work?

Event-driven architecture is made up of event producers and event consumers. An event producer detects or senses an event and represents the event as a message. It does not know the consumer of the event, or the outcome of an event. 

After an event has been detected, it is transmitted from the event producer to the event consumers through event channels, where an event processing platform processes the event _asynchronously_. Event consumers need to be informed when an event has occurred. They might process the event or may only be impacted by it. 

The event processing platform will execute the correct response to an event and send the activity downstream to the right consumers. This downstream activity is where the outcome of an event is seen.

**In conclusion** : The components of an event-driven architecture can include three parts: _producer_, _consumer_, _broker_. The broker can be optional, particularly when you have a single producer and a single consumer that are in direct communication with each other and the producer just sends the events to the consumer. An example would be a producer that is sending only to a database or data warehouse so the events are collected and stored for analysis. Most commonly in enterprises, you have multiple sources sending out all types of events with one or more consumers interested in some or all of those events.

## Nginx Architecture 

#### Overview of nginx Architecture

Traditional process- or _thread-based_ models of handling _concurrent connections_ involve handling each connection with a separate process or thread, and blocking on network or input/output operations. Depending on the application, it can be very inefficient in terms of memory and CPU consumption. Spawning a separate process or thread requires preparation of a new runtime environment, including allocation of heap and stack memory, and the creation of a new execution context. Additional CPU time is also spent creating these items, which can eventually lead to poor performance due to thread thrashing on excessive context switching. All of these complications manifest themselves in older web server architectures like Apache's. This is a tradeoff between offering a rich set of generally applicable features and optimized usage of server resources.

From the very beginning, nginx was meant to be a specialized tool to achieve more performance, density and economical use of server resources while enabling dynamic growth of a website, so it has followed a different model. It was actually inspired by the ongoing development of advanced event-based mechanisms in a variety of operating systems. What resulted is a _modular_, _event-driven_, _asynchronous_, _single-threaded_, _non-blocking_ architecture which became the foundation of nginx code (NGINX uses the _master-slave_ design).

nginx uses _multiplexing_ and _event notifications_ heavily, and dedicates specific tasks to separate processes. Connections are processed in a highly efficient run-loop in a limited number of single-threaded processes called `worker`s. Within each `worker` nginx can handle many thousands of concurrent connections and requests per second.

#### Core Structure

NGINX adheres to the master-slave design in the following : 

- **Masters** read and validate configurations by creating, binding, and crossing sockets. They also handle starting, terminations, and maintaining the number of configured worker processes (s_laves_). The master node can also reconfigure the worker process with no service interruption.(reading configuration and binding to ports..)
- **Workers** accept new requests from a shared listen socket and execute highly efficient run loops inside each worker to process thousands of requests. They do all of the work! They handle network connections, read and write content to disk, and communicate with upstream servers.
- **Proxy caches** are special processes. They have a _cache loader_ and _manager_. 
	- The **cache loader** checks the disk cache item and populates the engine’s in-memory database with the cache metadata. It prepares the NGINX instances to work with the files already stored on the disk in a specifically allocated structure. Runs at startup to load the disk‑based cache into memory, and then exits. It is scheduled conservatively, so its resource demands are low.
	- The **cache manager** handles cache expiration and invalidation. Runs periodically and prunes entries from the disk caches to keep them within the configured sizes.

In most cases, the recommended NGINX configuration of one worker process per CPU core makes the most efficient use of hardware resources. It can be customized by setting the worker_processes directive in NGINX configuration.

When the NGINX server is active, only the worker process is busy. Each worker process handles multiple connections in a non-blocking manner, reducing the number of context switches.

Each worker process is single-threaded and runs independently to acquire and process new connections. Processes can communicate using shared memory to obtain shared cache data, session persistent data, and other shared resources.

<p align="center">
<img src="./images/nginx_ps_model.png"/>
</p>
