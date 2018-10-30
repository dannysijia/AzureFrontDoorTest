# AzureFrontDoorTest

Azure Front Door Service enables you to define, manage, and monitor the global routing for your web traffic by optimizing for best performance and instant global failover for high availability.

For multinational customers, if website is located in US, the end users are at Asia, the performance may be very bad due to high latency and packet loss.

Front Door works at Layer 7 or HTTP/HTTPS layer and uses anycast protocol with split TCP and Microsoft's global network for improving global connectivity. So, per your routing method selection in the configuration, you can ensure that Front Door is routing your client requests to the fastest and most available application backend. An application backend is any Internet-facing service hosted inside or outside of Azure. Front Door provides a range of traffic-routing methods and backend health monitoring options to suit different application needs and automatic failover models.

## Test topology

We setup a AFD service at East Asia region. We build backend application pool at US East region.
We setup a small website with Apache/PHP/MySQL, test both HTTP and HTTPS performance from mainland China (Beijing).
Testing method include Chrome developer tools for single test and Apache benchmark for load testing.

## Building a backend server

We build a simple website in US east region with both HTTP and HTTPS capability. This server have static page and dynamic page to modify the webpage. This server have DNS name(yinghlisimplewebsite.eastus.cloudapp.azure.com) and map to a static IP(23.100.24.170).

I test this webserver from my laptop(123.121.197.100) with Psping and Apache bench.

```
Psping from my laptop to website, end to end latency is around 270ms.
C:\windows\system32>psping.exe 23.100.24.170:443

PsPing v2.10 - PsPing - ping, latency, bandwidth measurement utility
Copyright (C) 2012-2016 Mark Russinovich
Sysinternals - www.sysinternals.com

TCP connect to 23.100.24.170:443:
5 iterations (warmup 1) ping test:
Connecting to 23.100.24.170:443 (warmup): from 192.168.199.112:7627: 268.48ms
Connecting to 23.100.24.170:443: from 192.168.199.112:7628: 264.88ms
Connecting to 23.100.24.170:443: from 192.168.199.112:7629: 273.23ms
Connecting to 23.100.24.170:443: from 192.168.199.112:7633: 259.01ms
Connecting to 23.100.24.170:443: from 192.168.199.112:7634: 283.29ms

TCP connect statistics for 23.100.24.170:443:
  Sent = 4, Received = 4, Lost = 0 (0% loss),
  Minimum = 259.01ms, Maximum = 283.29ms, Average = 270.10ms
```

From my Chrome browser, using developer tools, I track the webpage loading performance.
![Chrome](https://github.com/yinghli/AzureFrontDoorTest/blob/master/detail.PNG)

We also use Apache Bench to test our website.
![ApacheBench](https://github.com/yinghli/AzureFrontDoorTest/blob/master/abtest.png)

Base on the performance report, we saw two challenges here. One is TCP connection setup time. This number is high due to high network latency. The other is page load time. This number is also impact by the network latency and page size. If we wants to improve the end use web&app experience, we need to focus on those two parts.

By design, AFD try to solve those two problem. We setup a front end as near as customer and terminate the connection. Front end reinitialize this connection to backend server, working as reverse proxy server. In this case, front end server and backend server are all inside Microsoft network, Microsoft backbone network can ensure that this connection is transport by a low latency and low packet drop network. Besides that, AFD can also provide caching function to reduce static information transfer time.

![splittcp](https://github.com/yinghli/AzureFrontDoorTest/blob/master/splittcp.png)

We setup AFD at Azure East Asia region. Front end name is "yinghli.azurefd.net".
![Frontend](https://github.com/yinghli/AzureFrontDoorTest/blob/master/frontend.PNG)

Add current webserver into backend pool. Setup HTTPS as health probes. Keep load balance rule as default.
![Backend](https://github.com/yinghli/AzureFrontDoorTest/blob/master/backend.PNG)

Setup default routing rule is to match any information under website and accept HTTP and HTTPS. 
![routing](https://github.com/yinghli/AzureFrontDoorTest/blob/master/routing.PNG)

In advance page, we enable caching and dynamic compression.
![routing2](https://github.com/yinghli/AzureFrontDoorTest/blob/master/routing2.PNG)

## Anycast

For more information about [Anycast](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-routing-architecture).
In my case, after setup AFD, I do a nslookup at my laptop. 

```
C:\windows\system32>nslookup yinghli.azurefd.net
Server:  Hiwifi.lan
Address:  192.168.199.1

Non-authoritative answer:
Name:    standard.t-0001.t-msedge.net
Addresses:  2620:1ec:bdf::10
          13.107.246.10
Aliases:  yinghli.azurefd.net
          t-0001.t-msedge.net
          Edge-Prod-HK2r3.ctrl.t-0001.t-msedge.net
```

## Round-Trip time

We measure the end to end latency for better understand the AFD.

```
C:\windows\system32>psping.exe yinghli.azurefd.net:443

PsPing v2.10 - PsPing - ping, latency, bandwidth measurement utility
Copyright (C) 2012-2016 Mark Russinovich
Sysinternals - www.sysinternals.com

TCP connect to 13.107.246.10:443:
5 iterations (warmup 1) ping test:
Connecting to 13.107.246.10:443 (warmup): from 192.168.199.112:16765: 42.31ms
Connecting to 13.107.246.10:443: from 192.168.199.112:16768: 44.77ms
Connecting to 13.107.246.10:443: from 192.168.199.112:16770: 45.61ms
Connecting to 13.107.246.10:443: from 192.168.199.112:16774: 45.27ms
Connecting to 13.107.246.10:443: from 192.168.199.112:16777: 44.49ms

TCP connect statistics for 13.107.246.10:443:
  Sent = 4, Received = 4, Lost = 0 (0% loss),
  Minimum = 44.49ms, Maximum = 45.61ms, Average = 45.04ms
```

Psping from my host to front end server, latency is 45ms. This is because I don't need to talk with real webserver in US East, instead of front end server will reply my information from Hong Kong.

From my Chrome browser, using developer tools, I track the webpage loading performance. From the AFD result, you can see that both connection and request/response time are shorter than before.
![Chrome](https://github.com/yinghli/AzureFrontDoorTest/blob/master/afdchrome.PNG)

## TCP connection on host

On my backend server, I can see 100+ TCP connection is setup from front end server backend.
```
root@USEastVM01:/home/sysadmin# netstat -anotp | grep "TIME_WAIT"
tcp6       0      0 10.0.1.4:443            147.243.0.13:42696      TIME_WAIT   -                    timewait (16.51/0/0)
tcp6       0      0 10.0.1.4:443            147.243.84.76:26251     TIME_WAIT   -                    timewait (21.72/0/0)
tcp6       0      0 10.0.1.4:443            147.243.156.76:48085    TIME_WAIT   -                    timewait (13.66/0/0)
tcp6       0      0 10.0.1.4:443            147.243.156.77:59123    TIME_WAIT   -                    timewait (40.67/0/0)
tcp6       0      0 10.0.1.4:443            147.243.86.213:14378    TIME_WAIT   -                    timewait (55.70/0/0)
tcp6       0      0 10.0.1.4:443            147.243.71.13:58581     TIME_WAIT   -                    timewait (31.26/0/0)
tcp6       0      0 10.0.1.4:443            147.243.84.205:54551    TIME_WAIT   -                    timewait (37.13/0/0)
tcp6       0      0 10.0.1.4:443            147.243.84.242:20154    TIME_WAIT   -                    timewait (25.32/0/0)
tcp6       0      0 10.0.1.4:443            147.243.72.13:28003     TIME_WAIT   -                    timewait (15.88/0/0)
tcp6       0      0 10.0.1.4:443            147.243.143.176:16429   TIME_WAIT   -                    timewait (50.66/0/0)
tcp6       0      0 10.0.1.4:443            147.243.6.140:63666     TIME_WAIT   -                    timewait (4.17/0/0)
tcp6       0      0 10.0.1.4:443            147.243.71.46:35764     TIME_WAIT   -                    timewait (46.87/0/0)
tcp6       0      0 10.0.1.4:443            147.243.132.15:55608    TIME_WAIT   -                    timewait (36.93/0/0)
tcp6       0      0 10.0.1.4:443            147.243.5.180:28250     TIME_WAIT   -                    timewait (47.98/0/0)
tcp6       0      0 10.0.1.4:443            147.243.78.115:37737    TIME_WAIT   -                    timewait (51.34/0/0)
tcp6       0      0 10.0.1.4:443            147.243.70.179:12328    TIME_WAIT   -                    timewait (19.40/0/0)
tcp6       0      0 10.0.1.4:443            147.243.84.109:12150    TIME_WAIT   -                    timewait (50.88/0/0)
tcp6       0      0 10.0.1.4:443            147.243.4.244:48547     TIME_WAIT   -                    timewait (50.86/0/0)
tcp6       0      0 10.0.1.4:443            147.243.16.237:19744    TIME_WAIT   -                    timewait (35.79/0/0)
tcp6       0      0 10.0.1.4:443            147.243.148.46:42466    TIME_WAIT   -                    timewait (41.03/0/0)
tcp6       0      0 10.0.1.4:443            147.243.70.206:31842    TIME_WAIT   -                    timewait (52.31/0/0)
tcp6       0      0 10.0.1.4:443            147.243.128.18:21714    TIME_WAIT   -                    timewait (14.34/0/0)
tcp6       0      0 10.0.1.4:443            147.243.128.19:61894    TIME_WAIT   -                    timewait (41.36/0/0)
tcp6       0      0 10.0.1.4:443            147.243.5.179:39019     TIME_WAIT   -                    timewait (20.92/0/0)
tcp6       0      0 10.0.1.4:443            147.243.70.204:51396    TIME_WAIT   -                    timewait (0.00/0/0)
tcp6       0      0 10.0.1.4:443            147.243.3.16:45403      TIME_WAIT   -                    timewait (41.64/0/0)
...
```

Those connection will be used when request is from end users.

I run Apache Bench test with same pressure, 10 concurrent and total 100 connection.
![afdabtest](https://github.com/yinghli/AzureFrontDoorTest/blob/master/afdabtest.PNG)

Then we increase the 10X pressure for testing. The result is still very good.
![afdabtest2](https://github.com/yinghli/AzureFrontDoorTest/blob/master/afdabtest2.PNG)

Even the concurrent connection is 100 per second, front end server will handle all those connection. 
Backend server will only see few TCP connection is in "TCP-ESTABLISHED" state.

```
root@USEastVM01:/home/sysadmin# netstat -anotp | grep "ESTABLISHED"
tcp6       0    258 10.0.1.4:443            147.243.9.139:57115     ESTABLISHED 11675/apache2        on (0.55/0/0)
tcp6       0    258 10.0.1.4:443            147.243.81.242:21903    ESTABLISHED 11648/apache2        on (0.63/0/0)
```
This will save backend server TCP connection resource.