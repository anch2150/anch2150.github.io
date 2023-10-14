---
title:  "Use BGP to Build Highly Available WireGuard at Home"
categories:
  - Homelab
tags:
  - BGP
  - BFD
  - High availability
  - WireGuard
  - Network
  - Routing
  - Raspberry Pi
---

## Intro
I've been running WireGuard on a single Raspberry Pi 4b since long ago. My current setup uses router's built-in **port forward** to NAT all packets arriving at `51820` to raspi `10.0.5.1:51820`.

![image](/assets/img/2022-09-07-ha-wireguard-0.svg)

The whole setup runs well until I need to remotely restart the board, for whatever reason. As I added another raspi recently to my home lab, it's time to explore a proper HA setup.

## Architecture

### Potential solutions
With a second server, there are a few options:

1. **Static routes** Set up two port forwards in the router, each to a server. There is no automatic failover, though.
2. **Active/standby** Use `VRRP` to dynamically assign a VIP to the primary server. The secondary server will sit idle, though.
3. **Reverse proxy** For most users, it is an overkill to get a hardware load balancer for a simple VPN setup. Running software load balancer like `haproxy` or `nginx` on router might also presents some challenge.
4. **Dynamic routing** Use `BGP` or `OSPF` to dynamically route packets to all available servers.

I ruled out 1 and 2 as the downside is very obvious.

For option 3, **Health check** is the biggest issue. nginx supports [**active** and **passive** UDP health check](https://docs.nginx.com/nginx/admin-guide/load-balancer/udp-health-check/). **Active** won't work, as WireGuard does not respond to any packets not encrypted via its public key. Apparently hacking nginx to encrpyt its health probe packets is insane (not thinking if it's possible at all). **Passive** means nginx assumes the endpoint is dead if no communication in X seconds, which implies some down time, which equals bad user experience. Additionally, it's very inefficient because all reverse proxies (nginx, haproxy, etc.) work at L4 in this scenario.

Given WireGuard is UDP-based, connection state is not a concern. If the packet stream of one connection is not delivered in order, or even if deliverd to differnt servers, it's not a serious problem (more discussion below).

**BGP** serves exacly the purpose:
* **Fast failover** BGP has hold and keepalive timers. With BFD it is even faster. Most implementation will fail immediately if the interface goes down, too.
* **Efficient** Packets are forwarded (L3) instead of proxied (L4).
* **Load balance** It's possible to announce multiple routes with same weight (See [Equal-Cost Multi-Path (ECMP)](https://en.wikipedia.org/wiki/Equal-cost_multi-path_routing)).

P.S. OSPF also works. I will cover it in my next post.

### BGP in a nutshell
In a typical BGP setup, both WireGuard servers (`10.0.5.1/8` and `10.0.7.1/8`) hold the same Virtual IP (VIP) `192.168.20.1/24`. They all periodically announce the route `192.168.20.0/24` to the router (`10.0.0.1/8`) via BGP protocol. The router adds those routes to its routing table.

![image](/assets/img/2022-09-07-ha-wireguard-1.svg)

When a packet arrives from Internet:
1. As `192.168.20.1/24` is not held by any interfaces of the router, it needs to forward the packet.
2. The router then searches the routing table and finds the best route(s) `via 10.0.5.1/8` and `via 10.0.7.1/8`.
3. The router hashes the packet as it's ECMP routing, then send it to the next hop.

If server `10.0.7.1/8` fails, it won't be able to make periodical announcement. The router will then remove the stale route from routing table. In step 2, only one route `via 10.0.5.1/8` will exist.

![image](/assets/img/2022-09-07-ha-wireguard-2.svg)

When the working server (`10.0.5.1/8`) receives the packet from `eth0`:
1. As `192.168.20.1/24` is held by `wg0`, the server forwards the packet from `eth0` to `wg0`
2. The packet reaches its destination, then gets picked up by WireGuard on `51820`

There is certainly some down sides of this setup. As there are two servers with equally weighted routes, there is an increased chance of out-of-order delivery of packets. Mis-delivered UDP packets may cause retransmission of the tunnelled TCP packet, or even drop of connection. This issue is not introcuded by BGP though and should be largely mitigated by the very fast failover of BGP routes before any tunnelled connection times out or gets dropped. Unfortunately I don't have the necessary lab setup to test different stress scenarios.

## Implementation

### Install BIRD
I use [**BIRD**](https://bird.network.cz/?get_doc&f=bird.html&v=20) on router and servers to set up BGP (and BFD).

My raspis run Ubuntu 22.04, which provides BIRD 2.0.8. To install, simply run
```shell
apt install bird2
```

My router runs FreshTomato. Firstly install [Entware](https://github.com/Entware/Entware/wiki/Install-on-TomatoUSB-and-FreshTomato#entware-on-freshtomato-and-other-tomatousb-forks), then run
```shell
opkg install bird2 bird2c
```

### Router setup
The router daemon runs two BGP sessions with each server daemon. It should learn from BGP where to forward packets destined `192.168.20.0/24` and merge it to its routing table. Meanwhile, it should not announce any other routes via BGP.

Below is the `/opt/etc/bird.conf` on router (assuming Entware is installed at `/opt`):
```conf
router id 10.0.0.1;                      # Use router LAN IP

protocol device {
    scan time 10;                        # Scan interfaces every 10 seconds
}

# Disable automatically generating direct routes to all network interfaces.
protocol direct {
    disabled;                            # Disable by default
}

# Synchronizing BIRD routing tables with kernel.
protocol kernel {
    ipv4 {                               # Connect protocol to IPv4 table by channel
        import none;                     # Import nothing from kernel
        export where source = RTS_BGP;   # Export BGP routes from bird table to kernel
    };
    merge paths yes;                     # Enable ECMP
}

# BGP peers
protocol bgp bgp_server5 {
    local as 65001;                      # Use a private AS number
    neighbor 10.0.5.1 as 65000;          # Peer (Server) IP and AS. The router is not part of the
                                         # WireGuard servers, therefore use a different AS.
    ipv4 {
        import all;                      # Import all from BGP
        export where source = RTS_BGP;   # Export BGP routes to bird table
    };
    bfd graceful;                        # Use BFD for faster liveness and failure detection
    graceful restart on;                 # Enable graceful restart
}

# Similar to bgp_server5
protocol bgp bgp_server7 {
    local as 65001;
    neighbor 10.0.7.1 as 65000;          # <--- Need to change the peer IP, but keep the SAME AS
    ipv4 {
        import all;
        export where source = RTS_BGP;
    };
    bfd on;
    graceful restart on;
}

# BFD sessions
protocol bfd {
    interface "br0";                     # Which interface to run BFD
    neighbor 10.0.5.1 dev "br0";         # Peer IP same as in BGP block, also only use interface "br0"
    neighbor 10.0.7.1 dev "br0";
}
```

### Server setup
Each server daemon only runs one BGP session with the router. It should announce route `192.168.20.0/24` to the router. Meanwhile, it does not need to import any other routes via BGP.

Below is the `/etc/bird/bird.conf` on server `10.0.5.1/24`:
```conf
log syslog all;

router id 10.0.5.1;

protocol device {
    scan time 10;                          # Scan interfaces every 10 seconds
}

# Disable automatically generating direct routes to all network interfaces.
protocol direct {
    disabled;                              # Disable by default
}

# Forbid synchronizing BIRD routing tables with the OS kernel.
protocol kernel {
    ipv4 {                                 # Connect protocol to IPv4 table by channel
        import none;                       # Import to table, default is import all
        export none;                       # Export to protocol. default is export none
    };
}

# Static IPv4 routes.
protocol static {
    ipv4;
    route 192.168.20.0/24 via "wg0";       # Set up the static route to WireGuard
}

# BGP peer
protocol bgp bgp_router {
    local as 65000;                        # Use private AS number, same as in router config
    neighbor 10.0.0.1 as 65001;            # The router's IP and AS, same as in router config
    ipv4 {
        import none;                       # Do not learn from BGP
        export where source = RTS_STATIC;  # Export only static routes.
    };
    bfd graceful;                          # Same as router
    graceful restart on;                   # Same as router
}

# BFD session
protocol bfd bfd_router {
    interface "eth0";                      # Which interface to run BFD
    neighbor 10.0.0.1;                     # Peer IP same as in BGP block
}
```

### Adjust port forward on router
Lastly, double check port forward. In my previous setup, the router forwards packets to `10.0.5.1:51820`. It should now forward to the new VIP `192.168.20.1:51820` instead.

## Verification

### Router outputs

Firstly check BFD sessions with `birdc`. Connections to both servers should be `Up`.
```console
# birdc show bfd sessions
BIRD 2.10.1 ready.
bfd1:
IP address                Interface  State      Since         Interval  Timeout
10.0.5.1                  br0        Up         21:26:38.754    0.100    0.500
10.0.7.1                  br0        Up         21:26:38.715    0.100    0.500
```

Then check BGP sessions with `birdc`. The stauts should show `Established`. There should be at least 1 route imported and exported.
```console
# birdc show protocols sessions
... (omitted for brevity)
bgp_server5 BGP        ---        up     2022-09-07    Established   # <--- Status is established 
  BGP state:          Established
    Neighbor address: 10.0.0.5
    Neighbor AS:      65000
    Local AS:         65001
    Neighbor ID:      10.0.0.5
    Local capabilities
      ... (omitted for brevity)
    Neighbor capabilities
      ... (omitted for brevity)
    Session:          external AS4
    Source address:   10.0.0.1
    Hold timer:       148.033/240
    Keepalive timer:  57.403/80
  Channel ipv4
    State:          UP
    Table:          master4
    Preference:     100
    Input filter:   ACCEPT
    Output filter:  (unnamed)
    Routes:         1 imported, 1 exported, 0 preferred
    Route change stats:     received   rejected   filtered    ignored   accepted  # <--- Route is imported and exported
      Import updates:              1          0          0          0          1
      Import withdraws:            0          0        ---          0          0
      Export updates:              1          0          0        ---          1
      Export withdraws:            0        ---        ---        ---          0
    BGP Next hop:   10.0.0.1

### bgp_server7 block should look almost identical
... (omitted for brevity)
```

Lastly check the kernel routing table. Notice that because we set `merge paths yes;`, both paths are merged with equal weights, i.e. `ECMP`.
```console
# ip route
...
192.168.20.0/24  proto bird  metric 32		# <-- Notice here protocol is bird (via BGP)
	nexthop via 10.0.5.1  dev br0 weight 1	# <-- Merged paths
	nexthop via 10.0.7.1  dev br0 weight 1	# <-- Merged paths
...
```

### Server
Also check BFD with `birdc`
```console
BIRD 2.0.8 ready.
bfd_router:
IP address                Interface  State      Since         Interval  Timeout
10.0.0.1                  eth0       Up         21:26:39.513    0.100    0.500
```

Then BGP with `birdc`
```console
... (omitted for brevity)
static1    Static     master4    up     2023-10-08    
  Channel ipv4
    State:          UP
    Table:          master4
    Preference:     200
    Input filter:   ACCEPT
    Output filter:  REJECT
    Routes:         1 imported, 0 exported, 1 preferred
    Route change stats:     received   rejected   filtered    ignored   accepted
      Import updates:              1          0          0          0          1  # <--- The static route is imported
      Import withdraws:            0          0        ---          0          0
      Export updates:              0          0          0        ---          0
      Export withdraws:            0        ---        ---        ---          0

bgp_r7000  BGP        ---        up     2023-10-09    Established   
  Description:    R7000
  BGP state:          Established
    Neighbor address: 10.0.0.1
    Neighbor AS:      65001
    Local AS:         65000
    Neighbor ID:      10.0.0.1
    Local capabilities
      ... (omitted for brevity)
    Neighbor capabilities
      ... (omitted for brevity)
    Session:          external AS4
    Source address:   10.0.7.1
    Hold timer:       224.985/240
    Keepalive timer:  68.356/80
  Channel ipv4
    State:          UP
    Table:          master4
    Preference:     100
    Input filter:   REJECT
    Output filter:  (unnamed)
    Routes:         0 imported, 1 exported, 0 preferred
    Route change stats:     received   rejected   filtered    ignored   accepted
      Import updates:              2          0          2          0          0  # <--- Nothing imported from BGP
      Import withdraws:            1          0        ---          1          0
      Export updates:              1          0          0        ---          1  # <--- The static route is exported
      Export withdraws:            0        ---        ---        ---          0
    BGP Next hop:   10.0.7.1
```

Lastly check routing table. Given we did not export anything from bird to kernel, we should only see the route created by default from interface address.
```console
# ip route
... (omitted for brevity)
192.168.20.0/24 dev wg0 proto kernel scope link src 192.168.20.1   # <-- notice here the protocl is kernel
```

## References
1. BIRD user guide: <https://bird.network.cz/?get_doc&v=20&f=bird-6.html#ss6.4>
2. cilium: <https://docs.cilium.io/en/stable/network/bird/>
