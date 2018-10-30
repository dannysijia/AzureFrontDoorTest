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

Psping from my laptop to website, end to end latency is around 270ms.
![Network](https://github.com/yinghli/AzureFrontDoorTest/blob/master/end2endlatency.png)

From my Chrome browser, using developer tools, I track the webpage loading performance.
![Chrome](https://github.com/yinghli/AzureFrontDoorTest/blob/master/detail.PNG)

We also use Apache Bench to test our website.
![ApacheBench](https://github.com/yinghli/AzureFrontDoorTest/blob/master/abtest.png)

Base on the performance report, we saw two challenges here. One is TCP connection setup time. This number is high due to high network latency. The other is page load time. This number is also impact by the network latency and page size. If we wants to improve the end use web&app experience, we need to focus on those two parts.

By design, AFD try to solve those two problem. We setup a front end as near as customer and terminate the connection. Front end reinitialize this connection to backend server, working as reverse proxy server. In this case, front end server and backend server are all inside Microsoft network, Microsoft backbone network can ensure that this connection is transport by a low latency and low packet drop network. Besides that, AFD can also provide caching function to reduce static information transfer time.

We setup AFD at Azure East Asia region. Front end name is "yinghli.azurefd.net".
![Frontend](https://github.com/yinghli/AzureFrontDoorTest/blob/master/frontend.PNG)

Add current webserver into backend pool. Setup HTTPS as health probes. Keep load balance rule as default.
![Backend](https://github.com/yinghli/AzureFrontDoorTest/blob/master/backend.PNG)

Setup default routing rule is to match any information under website and accept HTTP and HTTPS. 
![routing](https://github.com/yinghli/AzureFrontDoorTest/blob/master/routing.PNG)

In advance page, we enable caching.
![routing2](https://github.com/yinghli/AzureFrontDoorTest/blob/master/routing2.PNG)

## Anycast

We use typical DNS name lookup for our website and AFD instance.

## TCP connection on host

We can observe the TCP connection in out backend servers.

## Round-Trip time

We measure the end to end latency for better understand the AFD.
