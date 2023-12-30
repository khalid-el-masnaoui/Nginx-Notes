Bench-marking the server is the process of generating metrics of the throughput, responsiveness and reliability of the application response. This is the precursor to server optimization since the generated metrics serve as as a **_baseline_** that can be used to know the effectiveness of any optimization done.


# Table Of Contents
- **[section1](#section1)**
-  **[section1](#section1)**
	-  **[section1](#section1)**
	-  **[section1](#section1)**


## Performance testing

#### Introduction 

Put the system under pressure to determine various quality attributes. These attributes can server multiple purposes such us : 

- Validating whether the application meet a criterion
- Validating whether the system can perform in extreme conditions 
- Determine application bottlenecks
- Performance tunning

Performance testing can be :

- **Load tests** help you understand how a system behaves under an expected load. Its aim is to know the largest user load that the system can handle with accepted performance metrics.
- **Stress tests** help you understand the upper limits of a system's capacity using a load beyond the expected maximum.
-  **Spike testing** is used to figure out how your system reacts when it’s suddenly met with a large spike in users.

In other words, stress tests help you determine how a system would behave under an extreme load, such as a DDoS attack, Slashdot effect, or other scenarios. The goal is more to determine a maximum limit than to identify bottlenecks. That way, you can be prepared for unexpected circumstances.

On the other hand, load tests are designed to ensure that you meet user expectations, such as service level agreement (SLA) promises. The goal is to ensure an acceptable overall user experience rather than try to break the application. It lets you confidently deploy new code.

The goal is to establish the following parameters: 


- **_Throughput_** : This is defined as the rate at which the server can serve content, that is, concurrent requests per seconds that a server can handle. This is usually defines the upper limits of the web server.
- **_Error rate_** : This defined as the ratio of non-successful requests to the total number of requests. The error may have occurred due to sever unavailability because of high load or due to network timeouts. The metric is used in conjunction with the throughput as it is required to have the minimum error rate with a higher throughput.   
- **_Response time_** : This is defined as the Total Elapsed time between submission of a request and receipt of the response. Response Times can be measured at different tiers for a given application i.e. Web Server Response Time, Application Server Response Time, Database Server Response Time, etc. Response Times can also be measured within a given tier across the various system components i.e. CPU Response Time, Memory Access Response, Disk Access Response Time, Network Transfer Response Time, etc.



```
	 The basic Response Time Equations are written as:
	 
	- Rt = Wt + St …………………….  [ Wt = Wait Time, St = Service Time ] 
	- Rt = Qt + St  ……………………. [ Qt = Queuing Time , St = Service Time]
```

#### Timeouts

When the server is under load, the average response time increases as the server is
busy responding to the large number of requests it has received. When the server has
optimizations, the test can be run again, and you can compare whether the average
response time has improved or not. But, how can you go about comparing the error
rate and the throughput? The two results may not show you any error rate or a small
change in the throughput. This is because the server responds to all requests—just
that it takes more time in certain instances

In order to compare the error rate and the throughput, we need to take care of the
response time. This can be done using the timeouts defined in HTTP. The HTTP
protocol defines the following two timeouts:

- **Connection timeout** : This is the time it takes to create a connection (socket)
to the web server
- **Response timeout** : This is the time it takes for the server to send back a
response.

For any request, if a timeout occurs, it is treated as a failure. Thus, while determining
the error rate and the throughput in different tests, the timeouts should remain the
same. Basically, when we run the test to determine a metric, the other factors should
remain the same; otherwise, the numbers are difficult to compare.

#### Baselines 
A baseline is defined as the accepted attributes that describe a system at a particular point in time. Thus, the baseline serves as a point of reference. The idea is to capture performance metrics after every change and determine their effectiveness by comparing them to the baseline. Changes can only be compared one at a time. While working with baselines, we need to make sure that all aspects except for the single change must remain the same. Thus, the new metrics data, after the change, when compared to the baseline data, will show whether the performance improves or declines.

**Note** : Always run performance tests on a machine other than the server under test. If you run them on the same machine, the numbers generated will be misleading.

**Note 2** : Performance tests executed on various tools cannot be compared directly with each other. The tools vary significantly in how they simulate load and how metrics are determined; thus, the results vary largely when compared with each other. But the trend generated by tests on one tool is comparable to a trend from another tool. If a tool demonstrates a decrease in performance, then the other tool should provide a similar result.

**Note 3**  : Load simulation opens sockets on the client side. The sockets, in turn, are treated as file descriptors. Thus, make sure there are enough file descriptors available on the box. The limit can be enhanced by the `ulimit -n` command or by changing the `security.limits` file.


## Generating metrics with Tools

There are many tools for load testing your application.

#### siege 

Siege is an open source regression test and benchmark utility. It can stress test a single URL with a user defined number of simulated users, or it can read many URLs into memory and stress them simultaneously.

Siege allows you to stress a web server with n number of users t number of times, where n and t are defined by the user. It records the duration time of the test as well as the duration of each single transaction. It reports the number of transactions, elapsed time, bytes transferred, response time, transaction rate, concurrency and the number of times the server responded OK, that is status code 200.

By default, Siege keeps running for quite a while and prints a lot of logging information. You will be required to press Ctrl + C to close the program. The program executes the default load and prints the performance results, as shown in the following code:
```log
Transactions: 4443 
hits Availability: 100.00 % 
Elapsed time: 149.94 secs 
Data transferred: 15.56 MB 
Response time: 0.01 secs 
Transaction rate: 29.63 trans/sec 
Throughput: 0.10 MB/sec 
Concurrency: 0.05 
Successful transactions: 4443 
Failed transactions: 0 
Longest transaction: 0.01 
Shortest transaction: 0.00


- Transactions is the number of server hits. In the example, 25 simulated users [ -c25 ] each hit the server with 10 repetitions [ -r10 ], a total of 250 transactions.

- Elapsed time is the duration of the entire siege test. This is measured from the time the user invokes siege until the last simulated user completes its transactions. Shown above, the test took 14.67 seconds to complete.

- Data transferred is the sum of data transferred to every siege simulated user. It includes the header information as well as content. Because it includes header information, the number reported by siege will be larger then the number reported by the server. In internet mode, which hits random URLs in a configuration file, this number is expected to vary from run to run.

- Response time is the average time it took to respond to each simulated user’s requests.

- Transaction rate is the average number of transactions the server was able to handle per second, in a nutshell: transactions divided by elapsed time.

- Throughput is the average number of bytes transferred every second from the server to all the simulated users.

- Concurrency is average number of simultaneous connections, a number which rises as server performance decreases.

- Successful transactions is the number of times the server returned a code less then 400. Accordingly, redirects are considered successful transactions.
```

A few interesting default parameters to look at are the following: 
- **Concurrent users -c**: This specifies the load executed 
- **Failures until abort -f**: This specifies the number of failures that will abort the execution 
- **Time to run -t**: This specifies the amount of out time 
- **Repetitions -r**: This specifies the iterations to perform 
- **Delay -d5**:  This define the delay between each user request
- **Resource file -f**: This specifies the default configuration file (Urls file , a simple plain text file `--reps=once`) 

**Note 1** : Other similar tools are `ab`(apache-benchamrk), `wrk` ...
**Note 2** : `Siege` and `ab` are single-threaded , while `wrk` is multi-threaded

#### K6

k6 is **a high-performing load testing tool, scriptable in JavaScript**.

The k6 client command line allows you to generate many users directly from the client. This means that the k6 client acts as a controller (component orchestrating the test ) and a load generator (component sending the traffic to your application under test).

The k6 solution supports out-of-the-box HTTP (1.1 and 2, grpc and WebSocket). Still, there are several extensions for MQTT, AMQP, Kafka, Mllp, Redis, generating stress on your database level and driving a headless browser.

k6 can generate results like the below :

```log
scenarios: (100.00%) 1 scenario, 500 max VUs, 1m0s max duration (incl. graceful stop):

* default: 500 looping VUs for 30s (gracefulStop: 30s)

  
  

running (0m30.5s), 000/500 VUs, 15000 complete and 0 interrupted iterations

default ✓ [======================================] 500 VUs 30s

  

data_received..................: 1.2 GB 39 MB/s

data_sent......................: 1.2 MB 39 kB/s

http_req_blocked...............: avg=492.12µs min=2.29µs med=5.64µs max=50.06ms p(90)=8.7µs p(95)=17.95µs

http_req_connecting............: avg=452.39µs min=0s med=0s max=49.96ms p(90)=0s p(95)=0s

http_req_duration..............: avg=10.07ms min=921.3µs med=2.93ms max=138.15ms p(90)=35.3ms p(95)=49.24ms

{ expected_response:true }...: avg=10.07ms min=921.3µs med=2.93ms max=138.15ms p(90)=35.3ms p(95)=49.24ms

http_req_failed................: 0.00% ✓ 0 ✗ 15000

http_req_receiving.............: avg=532.76µs min=131.8µs med=495.17µs max=19.35ms p(90)=741.11µs p(95)=818.23µs

http_req_sending...............: avg=113.89µs min=8.65µs med=22.06µs max=42.12ms p(90)=43.54µs p(95)=52.35µs

http_req_tls_handshaking.......: avg=0s min=0s med=0s max=0s p(90)=0s p(95)=0s

http_req_waiting...............: avg=9.42ms min=703.98µs med=2.32ms max=98.57ms p(90)=34.45ms p(95)=48.5ms

http_reqs......................: 15000 491.048934/s

iteration_duration.............: avg=1.01s min=1s med=1s max=1.15s p(90)=1.03s p(95)=1.05s

iterations.....................: 15000 491.048934/s

vus............................: 500 min=500 max=500

vus_max........................: 500 min=500 max=500
```

You can read more about this awesome tool by reading [the official documentations](https://k6.io/docs/)


**Note** : When doing performance testing with any tool, be aware of the testing the `ssl version` of your application