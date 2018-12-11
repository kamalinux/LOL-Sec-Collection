
## Web Architecture
> what is web architecture?  
:syringe: Learned and noted for everyone :syringe:

**:alien: Overview :alien:**


![enter image description here](https://cdn-images-1.medium.com/max/1600/1*K6M-x-6e39jMq_c-2xqZIQ.png)

**:notebook: DNS :notebook:**

 - Domain Name Server
 - backbone technology that makes the world wide web possible
 - provides lookup value from domain name to ip address

Example

1. user --(`example.com`)--> DNS resolver -> 
2. DNS resolver --(`example.com`)--> root server
3. root server --(go to `name server for .com TLD`> DNS resolver 
4. DNS resolver --(`example.com`)--> name server for .com TLD 
5. name server for .com TLD --(go to `route 53 server`--> DNS resolver
6. DNS resolver --(`example.com`)--> route 53 server 
7. route 53 server --(`192.168.x.x`)--> DNS resolver 
8. DNS resolver --(`192.168.x.x`)--> user 
9. user --(`example.com`)--> Web Server ( `example.com or 192.168.x.x`)
10. Web Server (`Web page for example.com`)--> user 


**:notebook: Load Balancer :notebook:**

 - a device that distributes network or application traffic across a cluster of servers
 - improves responsiveness and increases availability of applications
 - sits between the client and the server farm accepting incoming network and application traffic and distributing the traffic across multiple backend servers using various methods
 - Core load balancing capabilities
	 - Layer 4 (L4) load balancing - the ability to direct traffic based on data from network and transport layer protocols, such as IP address and TCP port
	 - Layer 7 (L7) load balancing and content switching – the ability to make routing decisions based on application layer data and attributes, such as HTTP header, uniform resource identifier, SSL session ID and HTML form data
	 - Global server load balancing (GSLB) - extends the core L4 and L7 capabilities so that they are applicable across geographically distributed server farms

Example

[ user1, user2, user3 ] --`client traffics`-> ( Internet ) --> [ ADC ] --`directs requests to availabe server in the pool`-> [ server1 :heavy_check_mark:, server 2 :heavy_check_mark:, server3 :x:, server4 :heavy_check_mark: ]

 - ADC = Application Delivery Controller

Todo!!!!!!



Have fun! :v: :v: :v:

---
**:muscle: References :muscle:**  

all resources listed here :point_right: 
 - https://aws.amazon.com/route53/what-is-dns/
 - https://engineering.videoblocks.com/web-architecture-101-a3224e126947
 - https://www.citrix.com/glossary/load-balancing.html

:snowman: Contributed and maintained by [Luna-](https://twitter.com/art0flunam00n)  
:snowflake: Powered by [Legion of LOL](http://location-href.com)

---
