---
layout: post
title: What the ARP?
---

So back in the Fall of last year, [Digital Ocean](https://www.digitalocean.com/) became the second largest VPS provider in the world offering cloud infrastructure services similar to [Amazon AWS](https://aws.amazon.com/). Shortly after, they announced a feature that got my team and I very excited. [Floating IPs](https://www.digitalocean.com/community/tutorials/how-to-create-a-floating-ip-on-digitalocean). They sound like Magic, but my latest discovery would disprove that claim. Almost Magic, most of the time, is a better way to describe the feature. A blurb from Digital Ocean about Floating IPs:

*"A DigitalOcean Floating IP is a publicly-accessible static IP address that can be assigned to one of your Droplets...A Floating IP can also be instantly remapped...This instant remapping capability grants you the ability to design and create [High Availability (HA)](https://www.digitalocean.com/community/tutorials/how-to-set-up-highly-available-haproxy-servers-with-keepalived-and-floating-ips-on-ubuntu-14-04) server infrastructures..."*

So we began using the Floating IP feature to eliminate single points of failure in our infrastructure. We put redundant [HA proxy](http://www.haproxy.org/) load balancers behind a Floating IP with [KeepAlived](http://www.keepalived.org/) constantly monitoring the health of the two load balancers. In the event of a failure with our load balancers, KeepAlived would execute a script using Digital Ocean's API to re-assign the Floating IP to our redundant back up load balancer.

Almost flawless.

Most of our infrastructure is in Digital Ocean's NYC3 Data Center. We've recently expanded into San Francisco, Germany, and Bangalore Data Centers to give our customers a better experience that are geographically located. We have an endpoint that lives behind an HA proxy backend that gets called in order to authenticate users of our application from each of the four data centers listed above. Yesterday morning, we were alerted that some users connecting from the NYC3 region were not able to login to our application. Then the fun started.

I began investigating by doing the typical outage triage stuff. You know, checking out logs and retarting everything. Nothing was working. Something like this happened intermittently one time when DNS got screwy, so I attempted flushing the DNS cache on the machine with `sudo service nscd restart`.

```
devin@vm03:~$ sudo service nscd restart
 * Restarting Name Service Cache Daemon nscd
```

Then I tried to telnet to the port with `telnet learn.co 443`. Still nothing.

```
devin@vm03:~$ telnet learn.co 443
Trying 45.55.127.xxx...
```

Even ping gave me nothing with `ping learn.co`.

```
devin@vm03:~$ ping learn.co
PING learn.co (45.55.127.xxx) 56(84) bytes of data.
```

It wasn't until I looked at the Digital Ocean Floating IP [Control Panel](https://cloud.digitalocean.com/networking/floating_ips), that I realized that something was amiss. Instead of our main load balancer, the learn.co domain was resolving to our backup load balancer indicating a successful failover event. Well... almost. Only this one host was having issues connecting to learn.co on port 443. Hmmm..

We ended up opening a support ticket with Digital Ocean. They responded letting us know there was a known bug in the way their Floating IPs were designed. As it turns out, Floating IPs sometimes act a little buggy after a failover when the source host is in the same data center as the destination host owning the Floating IP. The work-around was simply to ping the public IP address assigned to the eth0 interface (the non floating IP) of the destination from the source host. After that, everything just magically started working again, but why? I thought it was time to dispell the magic, so I dug a bit deeper and began disecting the Link Layer.

![](http://snsscooters.com/sns/wp-content/uploads/2014/04/AARP.png)

ARP, a [Link Layer](https://en.wikipedia.org/wiki/Link_layer) protocol, not to be confused with the Association of Retired People, stands for [Address Resolution Protocol](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) and keeps a mapping of logical addresses to physical addresses of it's neighboring devices. In other words, it maps the IP address of an interface to it's MAC address for each host or network device that is immediately reachable (with no [Network Layer](https://en.wikipedia.org/wiki/Network_layer) routing). Hosts in the same subnet are only 1 hop away from each other meaning there's was no Network Layer routing needed in between source and destination to resolve connectivity. I tested this out with `traceroute`.

Internal Traceroute within the same subnet: (1 hop)

```
devin@vm03:~$ traceroute lb04.fe.flatironschool.com
traceroute to lb04.fe.flatironschool.com (159.203.114.xxx), 30 hops max, 60 byte packets
```

External Traceroute between different data centers: (12 hops)

```
devin@vm05:~$ traceroute lb04.fe.flatironschool.com
traceroute to lb04.fe.flatironschool.com (159.203.114.xxx), 30 hops max, 60 byte packets
 1  104.131.143.xxx (104.131.143.xxx)  5.186 ms  5.286 ms  5.199 ms
 2  138.197.248.xxx (138.197.248.xxx)  0.245 ms 138.197.248.xxx (138.197.248.xxx)  18.636 ms  18.627 ms
 3  xe-0-4-0-17.r06.plalca01.us.bb.gin.ntt.net (129.250.203.xxx)  1.262 ms  1.274 ms xe-0-0-0-23.r05.plalca01.us.bb.gin.ntt.net (129.250.204.xxx)  1.244 ms
 4  ae-15.r01.snjsca04.us.bb.gin.ntt.net (129.250.5.xxx)  73.102 ms  74.547 ms ae-15.r02.snjsca04.us.bb.gin.ntt.net (129.250.4.xxx)  76.355 ms
 5  ae-11.r22.snjsca04.us.bb.gin.ntt.net (129.250.3.xxx)  1.775 ms ae-1.r22.snjsca04.us.bb.gin.ntt.net (129.250.3.xxx)  1.758 ms  1.921 ms
 6  ae-8.r21.chcgil09.us.bb.gin.ntt.net (129.250.5.xxx)  58.308 ms  58.438 ms  57.552 ms
 7  ae-4.r20.chcgil09.us.bb.gin.ntt.net (129.250.2.xxx)  57.233 ms  56.531 ms  58.121 ms
 8  ae-0.r25.nycmny01.us.bb.gin.ntt.net (129.250.2.xxx)  76.670 ms  74.483 ms  77.038 ms
 9  ae-2.r07.nycmny01.us.bb.gin.ntt.net (129.250.3.xxx)  73.087 ms  73.086 ms  75.042 ms
10  xe-0-9-0-17.r08.nycmny01.us.ce.gin.ntt.net (129.250.204.xxx)  73.468 ms xe-0-2-0-14.r07.nycmny01.us.ce.gin.ntt.net (129.250.204.xxx)  75.482 ms  75.412 ms
11  * * *
12  lb04.fe.flatironschool.com (159.203.114.xxx)  75.547 ms  75.539 ms  73.837 ms
```

You can see the clear difference when running `traceroute` between hosts that are not located in the same subnet or in the same data center for that matter. They will always have 2 or more hops with Network Layer routing in charge of figuring out the route to take. So it's starting to make sense why the known bug with Floating IPs would have different behavior between hosts in the same data center or subnet vs. hosts geographically located. If you run `arp -n` you can actually see which neighboring devices the host knows about including the IP address, MAC address, and interface. This is called the ARP table. Running `ip neighbor` will actually show you the status of each ARP entry, usually "REACHABLE" or "STALE".

```
devin@vm03:~$ arp -n | egrep 'Address|159.203.114.xxx'
Address                  HWtype  HWaddress           Flags Mask            Iface
159.203.114.xxx           ether   04:01:be:xx:xx:xx   C                     eth0
```

```
devin@vm03:~$ ip neighbor | grep 159.203.114.xxx
159.203.114.xxx dev eth0 lladdr 04:01:be:xx:xx:xx STALE
```

Okay, so now I can see how internal connectivity got a little wonky. When the load balancer failover occured and changed the Floating IP owner, the routes were updated at the Network Layer, but not at the Link Layer for the hosts located in the same subnet as the Floating IP owner. So pinging the public IP address assigned to the eth0 interface of the destination must have somehow invalidated and reset the ARP cache for the source entry. Let's see what happens to the ARP table when we do that.

```
devin@vm03:~$ ping 159.203.114.xxx
PING 159.203.114.xxx (159.203.114.xxx) 56(84) bytes of data.
64 bytes from 159.203.114.xxx: icmp_seq=1 ttl=64 time=0.572 ms
64 bytes from 159.203.114.xxx: icmp_seq=2 ttl=64 time=0.308 ms
64 bytes from 159.203.114.xxx: icmp_seq=3 ttl=64 time=0.282 ms
^C
--- 159.203.114.xxx ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.282/0.387/0.572/0.131 ms
```

```
devin@vm03:~$ ip neighbor | grep 159.203.114.xxx
159.203.114.xxx dev eth0 lladdr 04:01:be:xx:xx:xx DELAY
```

```
devin@vm03:~$ ip neighbor
159.203.114.xxx dev eth0 lladdr 04:01:be:xx:xx:xx REACHABLE
```

Still feels like jiggling the handle to get the door open, but understanding the edge case makes me feel a bit better about the scope of the impact and how to avoid it. Digital Ocean said they would be sure to let us know when they've implemented a fix. Until then, we just added `ping 159.203.114.xxx -c 1` to the very beginning of the Chef recipe running on that host every couple of minutes to automate the temporary work-around.
