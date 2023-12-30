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