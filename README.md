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

Psping from my laptop to website, end to end latency is around 260ms.
![](https://github.com/yinghli/AzureFrontDoorTest/blob/master/end2endlatency.png)

We setup AFD at Azure East Asia region. Front end name is "", and default routing rule is "".

## Anycast

We use typical DNS name lookup for our website and AFD instance.

## TCP connection on host

We can observe the TCP connection in out backend servers.

## Round-Trip time

We measure the end to end latency for better understand the AFD.
