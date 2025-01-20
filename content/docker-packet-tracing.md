+++
title = "surprisingly thorny docker packet tracing"
date = 2025-01-03
+++

When I sat down to write this, I ambitiously started off by trying to understand exactly how all packets go from a running docker container to the internet.

More specifically, I wanted to take a default Docker Desktop v.4.37.1 instllation for macOS and run the following:

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

And trace where these packets go.

I did a lot of packet tracing and I still don't feel like I understand the complete route at the packet level because there are some black boxes along the way. Instead this has turned into a blind exploration and I simply want to share some things I found interesting.

#### Docker runs in an VM

The first one and I'm a little embarrassed to admit this but apparently the Docker Engine runs in a VM on your macOS. When you run something like:

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

None of these interfaces will exist on your local host and likewise when you run a container in the default network, you're not bridging directly to local host. You have to traverse the Docker VM first. This brings me to next discovery.

#### Virtual Ethernet Device Pairs for each container

Containers that are run in the default "bridge" network use virtual ethernet interfaces to route traffic to the Docker VM. Like I mentioned once already, the containers have to route traffic to the Docker VM first before it can be routed publicly and they use virtual ethernet pairs to do this.


Let's start a default network container:

```bash
/ # echo $HOSTNAME
b31eaaba7445
skennedy@sophon$ docker run --rm -it alpine sh
```


Let's take a look at Docker VM interfaces:

```bash
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
You can see a new one was added:

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

The [veth]([https://man7.org/linux/man-pages/man4/veth.4.html) prefix is a giveaway that this interface is a virtual ethernet device. The application for containers is spelled out in the man pages:

> "veth device pairs are useful for combining the network facilities of the kernel together in interesting ways.  A particularly interesting use case is to place one end of a veth pair in one network namespace and the other end in another network namespace, thus allowing communication between network namespaces."

Right. Right. Containers get their own network namespace when they run. These containers need a way to route from their network namespace to the VM's network namespace. In other words, you have one virtual ethernet interface on the host and one on the container. Let's get more information on the virtual ethernet device in on the VM:

```bash
/ # ip link show veth8712c33
34: veth8712c33@if33: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 65535 qdisc noqueue master docker0 state UP
    link/ether d6:5e:66:25:98:da brd ff:ff:ff:ff:ff:ff
```

The command above gives you 2 critical pieces of information, the layer 2 link id (34) and the address of the link id of the container (@if33). Let's view the veth interface in the container. Let's start by looking at the container interfaces:

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

It looks like we're getting somewhere. The address of our VM veth interface link id was 34 and the layer 2 link id of this interface is 33, which matches veth8712c33@if33. It appears that these 2 are paired. How can I test this? Since it looks like eth0 is the interface that is paired, let's send a packet directly out of eth0:

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

Cool. This pair has been verified. What's also interesting is that this seems to happen for every container run in the default network:

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

OKAY. So we now know how packets are routed to the VM. Let's revisit some details of the ping.

#### vpnkit is the VMs TCP/IP stack

If you recall, the source IP of traffic coming from the container above was 172.17.0.2. Putting aside how the container gets assigned that IP, how does that packet get its source IP? Let's take a look at routes on the container:

```bash
/ # echo $HOSTNAME
b31eaaba7445
/ # ip route get 34.160.111.145
34.160.111.145 via 172.17.0.1 dev eth0  src 172.17.0.2
```
That routing rule reads "for traffic that is destined for 34.160.111.145, the next hop is 172.17.0.1 and uses the **eth0** device using the source IP 172.17.0.2. If you recall, when this packet goes through this interface, it automatically appears on the VM (because of veth pairs) and thus begins its routing there so let's see where that packet is headed next.

If you take a look at any veth interface:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # ip link show vethc81ca61
18: vethc81ca61@if17: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 65535 qdisc noqueue master docker0 state UP
    link/ether d2:38:3a:e7:e9:94 brd ff:ff:ff:ff:ff:ff
```

You'll see that docker0 is listed there. This means that this interface is a member of docker0 and typically that means docker0 is a bridge. Let's verify this:

```bash
/ # echo $HOSTNAME
docker-desktop
/ # brctl show docker0
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02427378d4b8       no              vethc81ca61
```

So docker0 acts as a bridge for all containers to the VM. I'm sure if I add another container, a new member of the docker0 bridge will be present:

```bash
/ # brctl show docker0
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02427378d4b8       no              vethbf720cd
                                                        vethc81ca61
```

Okay, great so we now have verified that the packet goes to docker0. Let's trace the packet there:

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

This looks good. We see the packet arrive on docker0, make a DNS request to another interface to get ifconfig.me's IP and then eventually get back an ICMP echo request.

At this point, I want to remind you that the destination of the packet was 34.160.111.145 not 172.17.0.1. It can be tricky to see the above and assume docekr0 is the interface that the packet went out to the internet on, but no. Let's view our routes again:

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

Okay so we now know this packet goes out of eth0 on the VM, but where to now? There are no other VM interfaces for this thing to go to. This must mean it gets routed to the host. I will save you some suspense here, but the only interface that sees this on your host is the current active interface you're using to connect to the internet. Mine is **en9**:

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

That empty box is something called vpnkit. From this [blog article](https://www.docker.com/blog/how-docker-desktop-networking-works-under-the-hood/), vpnkit is:

> "a TCP/IP stack written in OCaml on top of the network protocol libraries of the MirageOS Unikernel project"

This tool was created to avoid clashes between VPN network modifications and guest VM network modifications. In other words, this tool implements a separate TCP/IP stack for the VM without having to modify routes, firewall rules or any additional TCP/IP stack stettings on the host. This is why you don't see any packets routed through the host system.


The documentation found [here](https://github.com/moby/vpnkit/blob/master/docs/ethernet.md) explains a few things:
- How the Docker VM is connected to vpnkit on the host
- What happens to ethernet frames when they arrive in vpnkit
- The process of a TCP connection

**TL;DR;** The VM talks to vpnkit running on the host through a virtual network interface. When vpnkit on the host receives ethernet frames from the VM, vpnkit creates individual virtual tcp/ip endpoints that act as proxies for the VM (this is the magic). These tcp/ip endpoint proxies send `SOCK_STREAM` and `SOCK_DGRAM` system calls to sockets on the host such that the host only sees the outgoing TCP or UDP packets on some interface.

#### dockerd and vpnkit high level network services?

There are also some higher level network services that vpnkit and dockerd implement such as HTTP Proxy, Docker API proxy and DNS. We actually encountered one of these services earlier: DNS. If you recall our packet trace on **docker0**, there was a DNS query made:

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

Let's see where this goes:

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

and let's view the VM trace of services1:

```bash
/ # tcpdump -n -i services1 "port 53"
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on services1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
17:16:12.521728 IP 192.168.65.6.35799 > 192.168.65.7.53: 26543+ [1au] A? seankennedy.sh. (55)
17:16:13.039491 IP 192.168.65.7.53 > 192.168.65.6.35799: 26543 2/0/0 A 172.67.160.254, A 104.21.15.12 (92)
```

Okay. This makes things a little more obvious, but there's probably other sorts of traffic that is routed through this interface and I may explore this in other posts.

#### Packet route recap

Let's recap the packet path we've discovered given a ICMP packet and a default macOS Docker Desktop installation:

When you make a ping request, the packet does the following:

- Goes out **eth0** of the container over a virtual ethernet interface
- Appears on the peer endpoint **vethxxxxx** interface of the VM
- It arrives on the **docker0** bridge interface because **vethxxxxx** interface is a member of the **docker0** interface and verified this.
- This **docker0** interface routes the packet to the VM **eth0** interface connected via a virtual device on the VM to vpnkit running on macOS
- In vpnkit some magic happens:
    - The packet gets switched to a newly created TCP/IP endpoint created by the mirageOS TCP-IP stack
    - The mirageOS TCP-IP stack opens a socket and makes a `SOCK_STREAM` or `SOCK_DGRAM` system call
- The packet egresses from the default active interface of the host. For me, this was **en9**.
- The connection between the vpnkit switch and TCP/IP endpoint stays open for reuse from the container.







































