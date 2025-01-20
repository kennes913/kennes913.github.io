+++
title = "surprisingly thorny docker packet tracing"
date = 2025-01-03
+++

When I started writing this, my goal was to unravel exactly how packets travel from a running Docker container (Docker Desktop v4.37.1) to the internet. Specifically, I wanted to understand how packets make it to the internet when you run something like:

 ```bash
 skennedy@sophon$ docker run --rm -it alpine sh

/ #  ping -4 -c 5 ifconfig.me
PING ifconfig.me (34.160.111.145): 56 data bytes
64 bytes from 34.160.111.145: seq=0 ttl=63 time=8.856 ms
64 bytes from 34.160.111.145: seq=1 ttl=63 time=9.022 ms
64 bytes from 34.160.111.145: seq=2 ttl=63 time=8.923 ms
64 bytes from 34.160.111.145: seq=3 ttl=63 time=9.221 ms
64 bytes from 34.160.111.145: seq=4 ttl=63 time=9.197 ms
 ```

I spent a lot of time tracing packets and mostly understand their route from local container to the public internet, however, some parts of the process remain obfuscated from tracing tools. I want this post to be more of an exploratory dive into the docker TCP/IP stack and I’d like to share some things I discovered along the way.

#### Docker Runs in a VM

The first discovery—and I’ll admit, it’s a bit embarrassing—is that the Docker Engine actually runs in a VM on macOS. When you execute a command like:

```bash
docker run --rm -it --privileged --network=host alpine sh
```

You're not starting a container that's able to talk to your macOS host network stack, instead you're connected to the network stack of the Docker VM. You can verify this by taking a look at the interfaces in the container above:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:71:DA:BC:58
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          inet6 addr: fe80::42:71ff:feda:bc58/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:65535  Metric:1
          RX packets:5739 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10614 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:484985 (473.6 KiB)  TX bytes:104753255 (99.9 MiB)

eth0      Link encap:Ethernet  HWaddr D2:4A:6E:3A:D6:83
          inet addr:192.168.65.3  Bcast:192.168.65.255  Mask:255.255.255.0
          inet6 addr: fe80::d04a:6eff:fe3a:d683/64 Scope:Link
          inet6 addr: fdc4:f303:9324::3/64 Scope:Global
          UP BROADCAST RUNNING MULTICAST  MTU:65535  Metric:1
          RX packets:617415 errors:0 dropped:0 overruns:0 frame:0
          TX packets:640284 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:229138097 (218.5 MiB)  TX bytes:245956035 (234.5 MiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:62 errors:0 dropped:0 overruns:0 frame:0
          TX packets:62 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:4782 (4.6 KiB)  TX bytes:4782 (4.6 KiB)

services1 Link encap:Ethernet  HWaddr 3E:91:24:52:5D:0D
          inet addr:192.168.65.6  Bcast:0.0.0.0  Mask:255.255.255.255
          inet6 addr: fdc4:f303:9324::6/128 Scope:Global
          inet6 addr: fe80::3c91:24ff:fe52:5d0d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:530319 errors:0 dropped:0 overruns:0 frame:0
          TX packets:603256 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:238232206 (227.1 MiB)  TX bytes:45871027 (43.7 MiB)

```

None of these interfaces exist on your local host. Similarly, when you run a container on the default network (no `--network=host` flag), the container's traffic isn't routed directly to the host TCP/IP stack; instead, it's traffic must first pass through the Docker VM. This leads to my next discovery.

#### Virtual Ethernet Device Pairs

Containers running on the default 'bridge' network use virtual ethernet interfaces to route traffic to the Docker VM. As mentioned earlier, all container traffic is first routed to the Docker VM before being sent to the public network, with virtual Ethernet pairs facilitating this connection.


Let's view this by starting a container on the default network:

```bash
/ # echo $HOSTNAME
b31eaaba7445
skennedy@sophon$ docker run --rm -it alpine sh
```

And let's see what happens to the interfaces on the VM:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:71:DA:BC:58
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          inet6 addr: fe80::42:71ff:feda:bc58/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:65535  Metric:1
          RX packets:5884 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10986 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:493334 (481.7 KiB)  TX bytes:107255756 (102.2 MiB)

eth0      Link encap:Ethernet  HWaddr D2:4A:6E:3A:D6:83
          inet addr:192.168.65.3  Bcast:192.168.65.255  Mask:255.255.255.0
          inet6 addr: fe80::d04a:6eff:fe3a:d683/64 Scope:Link
          inet6 addr: fdc4:f303:9324::3/64 Scope:Global
          UP BROADCAST RUNNING MULTICAST  MTU:65535  Metric:1
          RX packets:628303 errors:0 dropped:0 overruns:0 frame:0
          TX packets:649926 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:232697975 (221.9 MiB)  TX bytes:248598907 (237.0 MiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:62 errors:0 dropped:0 overruns:0 frame:0
          TX packets:62 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:4782 (4.6 KiB)  TX bytes:4782 (4.6 KiB)

services1 Link encap:Ethernet  HWaddr 3E:91:24:52:5D:0D
          inet addr:192.168.65.6  Bcast:0.0.0.0  Mask:255.255.255.255
          inet6 addr: fdc4:f303:9324::6/128 Scope:Global
          inet6 addr: fe80::3c91:24ff:fe52:5d0d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:539089 errors:0 dropped:0 overruns:0 frame:0
          TX packets:613801 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:240814537 (229.6 MiB)  TX bytes:46733738 (44.5 MiB)

veth8712c33 Link encap:Ethernet  HWaddr D6:5E:66:25:98:DA
          inet6 addr: fe80::d45e:66ff:fe25:98da/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:65535  Metric:1
          RX packets:145 errors:0 dropped:0 overruns:0 frame:0
          TX packets:380 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:10379 (10.1 KiB)  TX bytes:2503289 (2.3 MiB)
```


Comparing this ifconfig output to the original output above, you can see a new interface was added:

```bash
/ # ifconfig
veth8712c33 Link encap:Ethernet  HWaddr D6:5E:66:25:98:DA
          inet6 addr: fe80::d45e:66ff:fe25:98da/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:65535  Metric:1
          RX packets:145 errors:0 dropped:0 overruns:0 frame:0
          TX packets:380 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:10379 (10.1 KiB)  TX bytes:2503289 (2.3 MiB)
```

This is related to the container we started up. Let me explain.

The [veth]([https://man7.org/linux/man-pages/man4/veth.4.html) prefix is a giveaway that this interface is a virtual ethernet device. The application of veth interfaces for the container to VM relationship is spelled out in the man pages:

> "veth device pairs are useful for combining the network facilities of the kernel together in interesting ways.  A particularly interesting use case is to place one end of a veth pair in one network namespace and the other end in another network namespace, thus allowing communication between network namespaces."

Right. Right. Containers get their own network namespace when they run. These containers need a way to route from their network namespace to the VM's network namespace. In other words, you have one virtual ethernet interface on the host and one on the container. Let's get more information on the virtual ethernet device on the VM:

```bash
/ # ip link show veth8712c33
34: veth8712c33@if33: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 65535 qdisc noqueue master docker0 state UP
    link/ether d6:5e:66:25:98:da brd ff:ff:ff:ff:ff:ff
```

The command above provides two key pieces of information: the veth interface index (34) and the container's interface index (@if33). Now, let’s examine the veth interface inside the container, starting with its network interfaces:

```bash
/ # echo $HOSTNAME
b31eaaba7445
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:65535  Metric:1
          RX packets:380 errors:0 dropped:0 overruns:0 frame:0
          TX packets:145 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2503289 (2.3 MiB)  TX bytes:10379 (10.1 KiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
/ # ip link show eth0
33: eth0@if34: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 65535 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
```

This makes sense. The address of our VM veth interface interface index was 34 and the interface index of this interface is 33, which matches veth8712c33@if33. It appears that these 2 are paired. Since it looks like **eth0** is the interface that is paired, let's send a packet directly out of eth0:

```bash
/ # echo $HOSTNAME
b31eaaba7445
/ # ping -4 -I eth0 -c 1 ifconfig.me
PING ifconfig.me (34.160.111.145): 56 data bytes
64 bytes from 34.160.111.145: seq=0 ttl=63 time=9.068 ms

--- ifconfig.me ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 9.068/9.068/9.068 ms
```

And now let's view the trace of those ICMP packets on the veth interface in the VM:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # tcpdump -n -i veth8712c33 -vvv "icmp"
tcpdump: listening on veth8712c33, link-type EN10MB (Ethernet), snapshot length 262144 bytes
02:49:41.893104 IP (tos 0x0, ttl 64, id 46727, offset 0, flags [DF], proto ICMP (1), length 84)
    172.17.0.2 > 34.160.111.145: ICMP echo request, id 14, seq 0, length 64
02:49:41.902602 IP (tos 0x0, ttl 63, id 32653, offset 0, flags [none], proto ICMP (1), length 84)
    34.160.111.145 > 172.17.0.2: ICMP echo reply, id 14, seq 0, length 64
```

I am tracing only ICMP packets using **tcpdump** on the VM endpoint (veth8712c33) of the virtual ethernet pair. You can see that the packet src IP comes through as 172.17.0.2 and is sending a ICMP echo request to 34.160.111.145 which is the resolved IP for ifconfig.me.

Cool. This pair has been verified. What's also interesting is that this seems to happen for every container run in the default network. Let's start another container:

```bash
skennedy@sophon$ docker run --rm -it alpine sh

/ # echo $HOSTNAME
bf1ff96d3c05
```

See the new veth interface created on the VM:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # ifconfig | grep veth
34: veth8712c33@if33: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 65535 qdisc noqueue master docker0 state UP
36: veth84af899@if35: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 65535 qdisc noqueue master docker0 state UP
```

This also makes sense. We now know how packets are initially routed to the VM. Let's revisit some details of the ping.

#### vpnkit is the VMs TCP/IP Stack

If you recall, the source IP of traffic coming from the container above was 172.17.0.2. Putting aside how the container gets assigned that IP, how does that packet get its source IP? Let's take a look at routes on the container:

```bash
/ # echo $HOSTNAME
b31eaaba7445
/ # ip route get 34.160.111.145
34.160.111.145 via 172.17.0.1 dev eth0  src 172.17.0.2
```
The routing rule specifies: 'For traffic destined for 34.160.111.145, use 172.17.0.1 as the next hop, route it through the eth0 device, and set the source IP to 172.17.0.2.' When the packet passes through this interface, it automatically appears on the Docker VM (thanks to the veth pairs). From there, the packet's journey continues—so let’s examine where it’s headed next, which first requires a little more context on the VM's veth interface for this container.

If you take a look at any veth interface on the VM:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # ip link show vethc81ca61
18: vethc81ca61@if17: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 65535 qdisc noqueue master docker0 state UP
    link/ether d2:38:3a:e7:e9:94 brd ff:ff:ff:ff:ff:ff
```

You'll see that docker0 is listed there. This means that this interface is a member of docker0 which indiciates that docker0 could be a bridge. Let's verify this:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # brctl show docker0
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02427378d4b8       no              vethc81ca61
```

Does docker0 act as a bridge for all containers to the VM? Let's add another container and check the members of the docker0 bridge:

```bash
/ # brctl show docker0
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02427378d4b8       no              vethbf720cd
                                                        vethc81ca61
```

This is all making a little more sense now. We have verified that the packet goes to docker0 on the VM. Let's trace the packets flowing through that interface:

```bash
/ # tcpdump -n -i docker0 -vvv
tcpdump: listening on docker0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
03:15:39.540707 IP (tos 0x0, ttl 11, id 4545, offset 0, flags [DF], proto UDP (17), length 46)
    172.17.0.2.46503 > 34.160.111.145.33467: [bad udp cksum 0x3e70 -> 0x6807!] UDP, length 18
03:15:47.741868 IP (tos 0x0, ttl 64, id 15415, offset 0, flags [DF], proto UDP (17), length 57)
    172.17.0.2.60790 > 192.168.65.7.53: [bad udp cksum 0xadf9 -> 0x5b2f!] 60664+ A? ifconfig.me. (29)
03:15:47.832941 IP (tos 0x0, ttl 63, id 44586, offset 0, flags [DF], proto UDP (17), length 84)
    192.168.65.7.53 > 172.17.0.2.60790: [bad udp cksum 0xae14 -> 0x3ae3!] 60664 q: A? ifconfig.me. 1/0/0 ifconfig.me. [5m24s] A 34.160.111.145 (56)
03:15:47.834376 IP (tos 0x0, ttl 64, id 18011, offset 0, flags [DF], proto ICMP (1), length 84)
    172.17.0.2 > 34.160.111.145: ICMP echo request, id 31, seq 0, length 64
03:15:47.842983 IP (tos 0x0, ttl 63, id 18095, offset 0, flags [none], proto ICMP (1), length 84)
    34.160.111.145 > 172.17.0.2: ICMP echo reply, id 31, seq 0, length 64
```

This looks promising. We see the packet arrive on docker0, make a DNS request through another interface to resolve ifconfig.me's IP, and eventually receive an ICMP echo request in response.

However, let me remind you: the packet’s destination was 34.160.111.145, not 172.17.0.1. It’s easy to mistakenly assume that docker0 is the interface used to send the packet to the internet—but that’s not the case. Let’s review our VM routes again to clarify:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # ip route show table all
default via 192.168.65.1 dev eth0 table 2
192.168.65.1 dev eth0 table 2 scope link
127.0.0.0/8 dev lo scope host
172.17.0.0/16 dev docker0 scope link  src 172.17.0.1
192.168.65.0/24 dev eth0 scope link
192.168.65.7 dev services1 scope link
...
```

Apparently the command above will show you the routes across all routing tables (vs. the main routing table) and as you can see, the default route for all traffic is an interface called **eth0**. There's now an eth0 on the container and eth0 on the VM. Let's trace the packets on the VM eth0:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # tcpdump -n -i eth0 -vvv | grep "34.160.111.145"
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    192.168.65.3 > 34.160.111.145: ICMP echo request, id 23, seq 0, length 64
    34.160.111.145 > 192.168.65.3: ICMP echo reply, id 23, seq 0, length 64
```

Now we know the packet exits through eth0 on the VM—but where does it go next? With no other VM interfaces for it to route to, it must be heading to the host. To save you some suspense: the only interface on your host that sees this packet is the active interface you’re using to connect to the internet. In my case, that’s **en9**:

```bash
my-mac-host:~ root# tcpdump -n -i en9 -vvv | grep 34.160.111.145
tcpdump: listening on en9, link-type EN10MB (Ethernet), snapshot length 524288 bytes
    10.0.4.37 > 34.160.111.145: ICMP echo request, id 24, seq 0, length 64
    34.160.111.145 > 10.0.4.37: ICMP echo reply, id 24, seq 0, length 6
```

This still doesn't explain how the packet traversed from the VM to my host and after checking every single interface on my host, I am certain it's not going through those as well so what gives? The packet literally does:

```bash
VM -- eth0 ---> [        ] ---> mac --- en9 --> Internet
```

This tool was designed to prevent conflicts between VPN network modifications and guest VM network changes. Essentially, it provides a separate TCP/IP stack for the VM without requiring changes to routes, firewall rules, or other TCP/IP settings on the host. This is why no packets appear to be routed through the host system.

The [documentation](https://github.com/moby/vpnkit/blob/master/docs/ethernet.md) clarifies several key points:

- How the Docker VM connects to vpnkit on the host
- What happens to Ethernet frames when they reach vpnkit
- The steps involved in establishing a TCP connection

TL;DR: The VM communicates with vpnkit, running on the host, through a virtual network interface. When vpnkit receives Ethernet frames from the VM, it creates individual virtual TCP/IP endpoints that act as proxies for the VM—this is the "magic." These TCP/IP endpoint proxies make SOCK_STREAM and SOCK_DGRAM system calls to sockets on the host, allowing the host to see only the outgoing TCP or UDP packets on its network interfaces.

#### dockerd and vpnkit High Level Network Services

vpnkit and dockerd also implement higher-level network services, including an HTTP proxy, Docker API proxy, and DNS. In fact, we encountered one of these services earlier: DNS. If you recall from our packet trace on docker0, there was a DNS query made:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # tcpdump -n -i docker0 -vvv
tcpdump: listening on docker0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
...
03:15:47.741868 IP (tos 0x0, ttl 64, id 15415, offset 0, flags [DF], proto UDP (17), length 57)
    172.17.0.2.60790 > 192.168.65.7.53: [bad udp cksum 0xadf9 -> 0x5b2f!] 60664+ A? ifconfig.me. (29)
03:15:47.832941 IP (tos 0x0, ttl 63, id 44586, offset 0, flags [DF], proto UDP (17), length 84)
    192.168.65.7.53 > 172.17.0.2.60790: [bad udp cksum 0xae14 -> 0x3ae3!] 60664 q: A? ifconfig.me. 1/0/0 ifconfig.me. [5m24s] A 34.160.111.145 (56)
...
```

Let's investigate where packets routed to 192.168.65.7 go by inspecting the associated interfaces:

```bash
/ # ip route get 192.168.65.7
192.168.65.7 dev services1  src 192.168.65.6
/ # ifconfig services1
services1 Link encap:Ethernet  HWaddr 4A:77:4E:59:3C:E6
          inet addr:192.168.65.6  Bcast:0.0.0.0  Mask:255.255.255.255
          inet6 addr: fdc4:f303:9324::6/128 Scope:Global
          inet6 addr: fe80::4877:4eff:fe59:3ce6/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:106725 errors:0 dropped:0 overruns:0 frame:0
          TX packets:127024 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:36678404 (34.9 MiB)  TX bytes:10714222 (10.2 MiB)
```

Let's run a small trace on this to get a sense for the activity here:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # tcpdump -n -i services1 -c 25
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on services1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:52:47.514697 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 4128669845:4128670003, ack 2936187522, win 507, options [nop,nop,TS val 73092122 ecr 3107398295], length 158
16:52:47.515301 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 158, win 4094, options [nop,nop,TS val 3107398323 ecr 73092122], length 0
16:52:47.597042 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 158:492, ack 1, win 507, options [nop,nop,TS val 73092204 ecr 3107398323], length 334
16:52:47.598058 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 492, win 4092, options [nop,nop,TS val 3107398406 ecr 73092204], length 0
16:52:47.701064 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 492:803, ack 1, win 507, options [nop,nop,TS val 73092308 ecr 3107398406], length 311
16:52:47.702315 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 803, win 4092, options [nop,nop,TS val 3107398510 ecr 73092308], length 0
16:52:47.804952 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 803:1114, ack 1, win 507, options [nop,nop,TS val 73092412 ecr 3107398510], length 311
16:52:47.805902 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 1114, win 4092, options [nop,nop,TS val 3107398614 ecr 73092412], length 0
16:52:47.910931 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 1114:1427, ack 1, win 507, options [nop,nop,TS val 73092518 ecr 3107398614], length 313
16:52:47.911876 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 1427, win 4089, options [nop,nop,TS val 3107398720 ecr 73092518], length 0
16:52:48.018656 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 1427:1741, ack 1, win 507, options [nop,nop,TS val 73092626 ecr 3107398720], length 314
16:52:48.019736 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 1741, win 4087, options [nop,nop,TS val 3107398828 ecr 73092626], length 0
16:52:48.127582 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 1741:2055, ack 1, win 507, options [nop,nop,TS val 73092735 ecr 3107398828], length 314
16:52:48.129052 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 2055, win 4092, options [nop,nop,TS val 3107398937 ecr 73092735], length 0
16:52:48.233885 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 2055:2369, ack 1, win 507, options [nop,nop,TS val 73092841 ecr 3107398937], length 314
16:52:48.234723 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 2369, win 4092, options [nop,nop,TS val 3107399043 ecr 73092841], length 0
16:52:48.342904 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 2369:2683, ack 1, win 507, options [nop,nop,TS val 73092950 ecr 3107399043], length 314
16:52:48.343856 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 2683, win 4092, options [nop,nop,TS val 3107399152 ecr 73092950], length 0
16:52:48.465822 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 2683:2997, ack 1, win 507, options [nop,nop,TS val 73093073 ecr 3107399152], length 314
16:52:48.466225 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 2997, win 4092, options [nop,nop,TS val 3107399274 ecr 73093073], length 0
16:52:48.558584 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 2997:3311, ack 1, win 507, options [nop,nop,TS val 73093166 ecr 3107399274], length 314
16:52:48.559547 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 3311, win 4092, options [nop,nop,TS val 3107399368 ecr 73093166], length 0
16:52:48.668599 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 3311:3625, ack 1, win 507, options [nop,nop,TS val 73093276 ecr 3107399368], length 314
16:52:48.670230 IP 192.168.65.1.50909 > 192.168.65.7.2375: Flags [.], ack 3625, win 4092, options [nop,nop,TS val 3107399478 ecr 73093276], length 0
16:52:48.771188 IP 192.168.65.7.2375 > 192.168.65.1.50909: Flags [P.], seq 3625:3939, ack 1, win 507, options [nop,nop,TS val 73093378 ecr 3107399478], length 314
25 packets captured
26 packets received by filter
0 packets dropped by kernel
```

There's a lot of traffic here and it's kind of hard to know exactly what it is, but a pattern is emerging. Some select traffic and in our case, a DNS request is routed to the services1 interface and then out through eth0 (192.168.65.1). Let's run a dig directly:

```bash
/ # echo $HOSTNAME
b4a497b75eb0
/ # dig seankennedy.sh

; <<>> DiG 9.18.32 <<>> seankennedy.sh
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29334
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;seankennedy.sh.                        IN      A

;; ANSWER SECTION:
seankennedy.sh.         377     IN      A       172.67.160.254
seankennedy.sh.         377     IN      A       104.21.15.12

;; Query time: 5 msec
;; SERVER: 192.168.65.7#53(192.168.65.7) (UDP)
;; WHEN: Mon Jan 20 17:16:45 UTC 2025
;; MSG SIZE  rcvd: 92
```

And a dig without a trace on the interface in question is useless so let's trace the serviecs1 interface for DNS traffic:

```bash
/ # tcpdump -n -i services1 "port 53"
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on services1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
17:16:12.521728 IP 192.168.65.6.35799 > 192.168.65.7.53: 26543+ [1au] A? seankennedy.sh. (55)
17:16:13.039491 IP 192.168.65.7.53 > 192.168.65.6.35799: 26543 2/0/0 A 172.67.160.254, A 104.21.15.12 (92)
```

Alright this makes a little more sense now. This is the only easy example I could come up with, but I bet you there's other traffic that will come through this interface related to the abovementioned services.

#### Packet route recap

Let’s recap the packet path we’ve uncovered for an ICMP packet in a default macOS Docker Desktop installation. When you issue a ping request, the packet follows these steps:

- **Exits the container via eth0:** The packet is sent through the container’s eth0 over a virtual Ethernet interface.
- **Appears on the VM’s peer endpoint (vethxxxxx):** The packet arrives at the peer endpoint interface (vethxxxxx) within the VM.
- **Reaches the docker0 bridge interface:** Since the vethxxxxx interface is part of the docker0 bridge, the packet is routed through docker0.
- **Routed to the VM’s eth0 interface:** The docker0 bridge forwards the packet to the VM’s eth0 interface, which connects via a virtual device to vpnkit running on macOS.
- **Processed by vpnkit:**
    - In vpnkit, the following transformations occur:
    - The packet is switched to a newly created TCP/IP endpoint by the MirageOS TCP/IP stack.
    - The MirageOS TCP/IP stack opens a socket and makes a SOCK_STREAM or SOCK_DGRAM system call.
    - The connection between the vpnkit switch and the TCP/IP endpoint remains open for future use by the container.
- **Exits through the host’s active network interface:** The packet leaves the host through the currently active network interface (for me, this was en9).

This step-by-step breakdown captures the packet’s journey, highlighting the role of each interface and vpnkit in handling traffic from the container to the internet.







































