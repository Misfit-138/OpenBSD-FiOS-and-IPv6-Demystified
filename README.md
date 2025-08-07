# Demystifying OpenBSD 7.7 Router/Firewalls with Verizon FiOS Dual Stack (IPv4 + IPv6)

![Example Image](openbsdmountain2.jpg)

## Overview

This guide describes a **proven method** to configure an OpenBSD 7.7-based router/firewall with **dual stack IPv4 and IPv6** using residential **Verizon FiOS**. It includes support for dynamic IPv6 prefix delegation, DNS advertisement to LAN clients, DNSSEC, root server querying, or DNS over TLS using `unbound` to forward to Google, and optionally, DNS blocklisting utilizing RPZ.

Additionally, it attempts to explain how all the relevant components work together, under the hood, to provide functionality.

## Why?

- Because IPv6 is new and mysterious to me, and in my pursuit of understanding it, I wanted to share something with the community that may be helpful.
- Because simplicity is beautiful; OpenBSD is beautiful. It excels as a firewall/router platform, offering correctness, security, elegance and transparency. It resists the trend to incorporate bloatware disguised as features, inefficiency disguised as modernity, and unnecessary complexity. 

This guide shares a working configuration to help others build a reliable OpenBSD-based network gateway using the default tools found in the `base` system.

After reading, you should have a working understanding of what your OpenBSD firewall/router is doing and how it does so.

## 🔧 OpenBSD `base` Tools Used

- OpenBSD 7.7
- `pf` (OpenBSD's Packet Filter)
- `dhcpd` (IPv4 DHCP server daemon)
- `dhcpleased` - (OpenBSD's IPv4 DHCP client daemon replacing the older ISC dhclient)
- `dhcp6leased` (OpenBSD's IPv6 prefix delegation client)
- `rad` (OpenBSD's Router Advertisement Daemon)
- `slaacd` (StateLess Address Automatic Configuration daemon)
- `unbound` (Validating, recursive, caching DNS resolver)
- `rpz` (Response Policy Zones- DNS filtering feature in `unbound` )

## Hardware requirements:

Your choice of hardware will be dictated by your own network requirements, but, generally, a machine with at least 2 NICs (one for WAN, one for LAN) that is of a supported architecture will work.
Intel NICs are particularly well supported.

# ⚠️ DISCLAIMER ⚠️

**READ THIS CAREFULLY BEFORE PROCEEDING**

This guide contains network security configurations that will control your firewall, routing, and DNS settings. **DO NOT blindly copy and paste these configurations without understanding what they do.**

**BEFORE USING THIS GUIDE:**
1. **Understand each configuration line** before applying it
2. **Test in a lab environment** first if possible
3. **Have console/physical access** to your router
4. **Keep backups** of working configurations
5. **Know how to recover** if something goes wrong

**Each network's requirements are different. Adapt accordingly and verify each setting matches your environment.**

**YOU ARE RESPONSIBLE** for understanding and securing your own network. Use this guide as reference material, not as a copy-paste solution.

---
# In this guide, ix1 is WAN, ix0 is LAN

## 📦 Installation Steps

## 1. Install OpenBSD 7.7
Official instructions at: https://www.openbsd.org/faq/faq4.html

During the install set up a WAN interface configured with automatic IPv4/IPv6 and a LAN interface with static IPv4 for now.

## 2. Check for *running* and *enabled* daemons 

```sh
rcctl ls started
rcctl ls on
```
`slaacd` **should** be enabled by default in a fresh install, but disable for now:
```sh
rcctl stop slaacd
rcctl disable slaacd
```

`dhcp6leased` might be running also.

Disable for now:
```sh
rcctl stop dhcp6leased 
rcctl disable dhcp6leased 
```

## 3. Disable `resolvd` (recommended for a router)  (IPv4/IPv6)
I would recommend having full control over DNS (to avoid ISP DNS being assigned to router via DHCP):

```sh
rcctl stop resolvd
rcctl disable resolvd
```
## 4. Edit configuration files
### `/etc/sysctl.conf`  (IPv4/IPv6)
Turning the system into a router that will forward IPv4/IPv6 packets between network interfaces is simple and easy:
```conf
# /etc/sysctl.conf for router/firewall
net.inet.ip.forwarding=1
net.inet6.ip6.forwarding=1
```
### `/etc/dhcpd.conf`  (IPv4)
Setting up an IPv4 DHCP server is also straightforward.

A simple, sane, working example:
```conf

# /etc/dhcpd.conf
subnet 192.168.1.0 netmask 255.255.255.0 {
	option routers 192.168.1.1;
	domain-name-servers 192.168.1.1;
	range 192.168.1.10 192.168.1.254;
}
```
### `/etc/resolv.conf`  (IPv4/IPv6)
This file configures the *router's* resolving behavior.
```conf
# /etc/resolv.conf
nameserver 127.0.0.1
nameserver ::1
lookup file bind
search home.arpa  # <--- replace with your local domain name
```
`nameserver 127.0.0.1`→ Use the local DNS server running on IPv4 loopback (localhost). This means the system will try to send DNS queries to itself at 127.0.0.1.

`nameserver ::1`→ Same as above, but for IPv6 loopback. This allows DNS queries to be sent to localhost using IPv6.

`lookup file bind`→ This controls the order of name resolution:

`file`→ means check /etc/hosts first.

`bind`→ Originally meant the **Berkeley Internet Name Domain (BIND)** software, which is the original and widely used DNS resolver and server suite developed at Berkeley in the early days of the Internet. It’s a legacy term that stuck. Now, it simply means *ask the DNS servers listed above (127.0.0.1 and ::1) if the name wasn’t found in /etc/hosts*.

`search home.arpa`→ If you type just a short hostname (like, myserver), the system will try appending .home.arpa to it, making it myserver.home.arpa. This helps with resolving names in your home network automatically *from the router itself*.


### Create `/etc/dhcpleased.conf`  (IPv4)

dhcpleased does the following:
- Runs on interface(s) configured with `inet autoconf` (`ix1` in our example)
- Sends DHCP Discover/Request messages to find and lease an IPv4 address.
- Receives offers from a DHCP server, selects one, and requests it.
- Writes lease info to /var/db/dhcpleased/ so it persists across reboots.
- Configures your interface with:
  - IPv4 address
  - Default gateway (route add default)
  - DNS resolvers (written to /etc/resolv.conf)
  - Renews leases as needed to keep the IP assignment active.

The following simple example will ignore ISP DNS assignment (recommended). Also, be sure to disable `resolvd`, (recommended above). This gives us full control over our DNS:
```conf
# /etc/dhcpleased.conf
interface ix1 { ignore dns }
```
`dhcpleased` is automatically started on interfaces marked with `inet autoconf` in `/etc/hostname.if`, so it should be enabled and running on `ix1` after a fresh install if you chose (autoconf). 

Restart with:
```sh
rcctl restart dhcpleased
```
At this point, you may wish to go back and re-check your resolv.conf, to ensure it has not been overwritten.
### Check WAN for public IPv4 address:
```sh
ifconfig ix1
```
You should see a valid IPv4 address.

### `/etc/dhcp6leased.conf`:  (IPv6)

`dhcp6leased` is based on `dhcpleased` and serves a similar purpose and function, but for IPv6. It runs on interface(s) configured with `inet6 autoconf`.

This simple file is all that is needed:
```conf
# /etc/dhcp6leased.conf
request prefix delegation on ix1 for { ix0/64 }
```
**(WAN interface: ix1, LAN interface: ix0)**
## Explaining this simple file with IPv6 Math
- IPv6 addresses are 128 bit.
- A *prefix* defines an IPv6 *subnet*.
- Verizon gives out /56 prefixes; The first 56 bits are *fixed* by Verizon.
- Each /64 subnet: The first 64 bits define the subnet. And, since the first 56 are fixed,
- Bits 56 to 64 = 64 - 56 = 8 bits, so,
- 8 bits are available for our own subnetting.
- With 8 bits, we can have 2^8 = 256 different values (0 through 255). This gives us 256 possible /64 subnets for our network.
- **A /64 prefix defines an IPv6 subnet** that contains 18 Quintillion addresses; Every address in those subnets is a GUA (Global Unicast Address)- Quite generous. 
- **The `/etc/dhcp6leased.conf` syntax does exactly what it implies: Request prefix delegation on WAN `ix1` and assign the first /64 to LAN `ix0`**
- The remaining 64 bits are client **SLAAC** territory.

## Breakdown: ##

56 bits: Fixed by Verizon (2006:4040:1234:56)

8 bits: Yours for subnetting (00, 01, 02... ff)

64 bits: Host addresses within each subnet (SLAAC territory)

Math check:
56 + 8 + 64 = 128 bits

What this gives you:
- 256 possible subnets (from your 8 bits: 2^8 = 256)
- Each subnet can have ~18 quintillion hosts (from the 64 host bits: 2^64)

Example subnet:

2006:4040:1234:5601::/64

2006:4040:1234:56 = Verizon's 56 bits

01 = Your subnet choice (1 of 256 possible)

:: = 64 zero bits available for host addresses

On that subnet, device SLAAC clients can assign:

2006:4040:1234:5601::1 (assigned by `slaacd` on OpenBSD router)

2006:4040:1234:5601::a1b2:c3ff:fed4:5678 (assigned by device 1 SLAAC client)

2006:4040:1234:5601::dead:beef:cafe:1234 (assigned by device 2 SLAAC client)

...etc.

So yes - we get 256 massive subnets, each capable of holding far more devices than exist on Earth.
# Every address in those subnets is a GUA (Global Unicast Address).
> 
> **Why they're all GUAs:**
> * **Global routing prefix**: `2006:4040::/32` is allocated to Verizon by IANA
> * **Globally routable**: Any address starting with `2006:4040:` can be reached from anywhere on the IPv6 internet
> * **Unique worldwide**: No other network uses your specific `/56` prefix
> 
> **So all of these are GUAs:**
> * `2006:4040:1234:5600::1` (router on subnet 0)
> * `2006:4040:1234:5601::dead:beef` (device on subnet 1)
> * `2006:4040:1234:56ff::a1b2:c3d4` (device on subnet 255)
> 
> **What makes them "global":**
> * They're not private/local addresses (like `fc00::/7` or `fe80::/10`)
> * They're not multicast (`ff00::/8`) or reserved ranges
> * They're part of the global IPv6 routing table
> 
> **The beauty of IPv6:** Unlike IPv4 where you typically get one public IP and NAT everything else, with IPv6 every device gets its own globally routable address. Your laptop, phone, IoT devices - they all get real internet addresses that can be reached directly (subject to firewall rules).
> 
> We have 256 subnets, each with ~18 quintillion globally unique, internet-routable addresses.
>

### `/etc/hostname.ix0` (LAN):  (IPv4/IPv6)

### Create a ULA (Unique Local Address)- IPv6's equivalent to RFC 1918 private addresses like 192.168.x.x in IPv4.
Why it's useful:

* **ULA**s provide stable, predictable addresses for local network communication.
* Unlike **GUA**s (which can change when your ISP changes your delegated prefix), **ULA**s remain constant.
* Ensures local services and device-to-device communication continues working even if your ISP prefix changes-Provides a fallback for local network services.

You *could* easily create your own random ULA/prefix using `jot`:
```sh
jot -r 6 0 255 | xargs printf "fd00:%02x%02x:%02x%02x:%02x%02x::1/64\n"
```
The output should be something like:
```sh
fd00:AAAA:BBBB:CCCC::1/64
```
But, ULAs are private and not routable on the internet, so feel free to arbitrarily create your own ULA which is more readable;
```fd00:0red:dead:0000::1/64``` is perfectly fine as well.

So, now we have:
- A ULA: fd00:AAAA:BBBB:CCCC::1 = ULA (the individual address)
- A subnet/prefix: fd00:AAAA:BBBB:CCCC::/64 = ULA prefix (the subnet)

Terminology clarification:

When we write out fd00:AAAA:BBBB:CCCC::1/64, we are specifying:
- The *individual ULA address*: fd00:AAAA:BBBB:CCCC::1 with subnet information: /64
- The *prefix* is: fd00:AAAA:BBBB:CCCC::/64

We will use the ULA as an alias for the LAN interface in `hostname.ix0`. The prefix will be used in `unbound.conf`, and Later, we will use **both the prefix and the ULA** to plug into `rad.conf` to advertise to our LAN clients. In this way, we have a permanent address on our LAN interface as well as a /64 subnet for private use that will not change, unlike the dynamic prefix and the Global Unicast Address (GUA) within from the ISP assigned to the LAN. *More on this later.*

```sh
# /etc/hostname.ix0 (LAN):
inet 192.168.1.1 255.255.255.0 192.168.1.255
inet6
inet6 alias fd00:AAAA:BBBB:CCCC::1/64  # ULA alias for LAN interface (Create your own.)
```
- `inet 192.168.1.1 255.255.255.0 192.168.1.255`:  
  Assigns a static IPv4 address to the interface, using a standard /24 subnet. Devices on the LAN will use this as their IPv4 gateway.

- `inet6`:  
  Enables IPv6 processing on the LAN interface without assigning a static IPv6 address. This line is essential for SLAAC and for receiving Router Advertisements (RAs) from `rad`.  
  Without this line, `slaacd` will not run on the interface, and the interface will not receive a dynamic Global Unicast Address (GUA) or an automatic default route.  
  With this line present, `slaacd` autoconfigures a GUA when RAs are received.

- `inet6 alias fd00:AAAA:BBBB:CCCC::1/64`:  
  Assigns a stable Unique Local Address (ULA) to the LAN interface. For use with internal-only services (like unbound or NTP), providing consistent local IPv6 reachability even if the delegated GUA prefix changes or is unavailable.

This simple setup ensures:
- Dual-stack (IPv4 + IPv6) support on the LAN
- Dynamic GUA assignment via SLAAC (enabled by `inet6`)
- A fixed local IPv6 address for internal services (via ULA)
  
### `/etc/hostname.ix1` (WAN) should *probably* be configured during install:  (IPv4/IPv6)
Simple, clean and brainless. And, it *just works*:
```sh
# /etc/hostname.ix1 (WAN)
inet autoconf
inet6 autoconf
```

- `inet autoconf`: Enables DHCPv4 on the WAN interface. The system will automatically obtain a public IPv4 address, subnet mask, and default gateway from the ISP via `dhcpleased`.
- `inet6 autoconf`: Enables IPv6 autoconfiguration on the WAN interface. This allows the router to obtain a link-local IPv6 address on the WAN and communicate with the ISP’s DHCPv6 server for prefix delegation via `dhcp6leased`.
  
### 🔥 `/etc/pf.conf` (Firewall Rules)  (IPv4/IPv6)

A clean and concise dual stack PF configuration with minimal logging, which works with both IPv4 and IPv6. It is based on a "block all in, let anything out" foundation, with security against spoofing, and selected filtering for functionality; this verbatim configuration is generally fine for a trusted home LAN, but again, KNOW WHAT YOU ARE DOING.

## *IPv6 essential considerations:*
- There must be a route-to or pass out rule in `pf.conf` for IPv6 outbound from LAN to WAN. This is easy to forget and will silently block IPv6. This is covered, because our example let's everything out- `pass out quick inet6 keep state`.
- ICMPv6 needs to be allowed on WAN and LAN or many things break- neighbor discovery, path MTU discovery, RA, etc. This is covered by `pass in quick inet6 proto ipv6-icmp from any to any icmp6-type {
    echoreq, echorep, unreach, toobig, timex, paramprob,
    neighbrsol, neighbradv, routersol, routeradv
} keep state`
- `dhcp6leased` <-> server traffic (udp port 546 <- 547) must be enabled on WAN ix1. Our example covers this with `pass in quick on egress inet6 proto udp from any port 547 to any port 546`.

# ⚠️ IMPORTANT SECURITY DISCLAIMER ⚠️

**THIS CODE DEFINES THE SYSTEM FIREWALL BEHAVIOR. USE AT YOUR OWN RISK.**

```pf
# /etc/pf.conf
# Interface macros
lan = "ix0"

# Martian addresses
table <martians> {
    0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16
    172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 192.168.0.0/16
    198.18.0.0/15 198.51.100.0/24 203.0.113.0/24 224.0.0.0/3
}
table <martians6> {
    ::/128 ::1/128 ::ffff:0:0/96 100::/64 2001::/23
    2001:2::/48 2001:10::/28 2001:db8::/32 fc00::/7 fec0::/10
}

set block-policy drop
set loginterface egress
set skip on lo

match in all scrub (no-df random-id max-mss 1440)

# NAT for IPv4
match out on egress inet from !(egress:network) to any nat-to (egress:0)

# Anti-spoofing
antispoof quick for { egress $lan }

# Block martian traffic
block in quick on egress inet from <martians> to any
block return out quick on egress inet from any to <martians>
block in quick on egress inet6 from <martians6> to any
block return out quick on egress inet6 from any to <martians6>

# Default deny
block log all

# Allow anything out
pass out quick inet keep state
pass out quick inet6 keep state

# Allow LAN clients
pass in on $lan keep state

# ICMP
# IPv4
pass in quick inet proto icmp from any to any icmp-type { echoreq, unreach } keep state
# IPv6
pass in quick inet6 proto ipv6-icmp from any to any icmp6-type {
    echoreq, echorep, unreach, toobig, timex, paramprob,
    neighbrsol, neighbradv, routersol, routeradv
} keep state

# DHCPv6 client
pass in quick on egress inet6 proto udp from any port 547 to any port 546
```
### Reload `pf` rules:  (IPv4/IPv6)

```sh
pfctl -f /etc/pf.conf
```

## 5. Acquire Delegated Prefix (IPv6)

### Enable and run `dhcp6leased` manually to observe prefix delegation

```sh
rcctl enable dhcp6leased
dhcp6leased -d
```
Recall from above that we configured `/etc/hostname.ix1` with `inet6 autoconf`. Therefore, `dhcp6leased` is called on `ix1`.

You should see something like:

```
...prefix delegation #1 2600:4040:AAAA:BBBB::/56 received on ix1 from server ...
```

Stop `dhcp6leased` (Ctrl+C) and start it normally:
```sh
rcctl start dhcp6leased
```
Having negotiated the lease, `dhcp6leased` writes the prefix to `/var/db/dhcp6leased/ix1`

## 6. 📡 Create `/etc/rad.conf` (Router Advertisement)  (IPv6)
Direct `rad` to advertise on `ix0` (LAN):
```conf
# /etc/rad.conf
interface ix0 {
    prefix fd00:AAAA:BBBB:CCCC::/64
    dns {
        nameserver fd00:AAAA:BBBB:CCCC::1
    }
}
```
*Substitute with your actual ULA and prefix created above.*

This configures `rad`  to advertise both the ULA *prefix* and *address* to your LAN:
- Clients will receive both the DNS address (fd00:AAAA:BBBB:CCCC::1) and the ULA prefix (fd00:AAAA:BBBB:CCCC::/64).
- They will autoconfigure ULA addresses like fd00:AAAA:BBBB:CCCC::abcd for themselves from the prefix using SLAAC.
- *They then use those ULA source addresses to query DNS* at your router’s ULA (fd00:AAAA:BBBB:CCCC::1).


We've essentially created a ULA subnet (fd00:AAAA:BBBB:CCCC::/64) on the LAN for the specific purpose of stable internal DNS service, (despite our upstream GUA prefix being dynamic and subject to change), and configured `unbound` to listen on the ULA address fd00:AAA:BBBB:CCCC::1

This concept was strange to me at first, since, coming from the frugality of IPv4, it seemed excessive to create 18 quintillion addresses simply for my little network's DNS. 

But, IPv6 actually encourages this for:
- Stability: Your ULA doesn’t change like your Verizon-assigned GUA. This makes it a perfect anchor for DNS, which needs consistency.
- Privacy: ULAs aren't routable on the public Internet, so there's no exposure.
- Simplicity: You avoid having to dynamically reconfigure Unbound or clients whenever your GUA changes.
- Reachability: Clients can always find Unbound at fd00:AAAA:BBBB:CCCC::1, even if your global prefix changes.

## But what about our Verizon delegated prefix? Don't we need to configure `rad` to advertise it, in `rad.conf`?

No. `rad.conf` does not need the delegated prefix to be explicitly included.

*As long as the prefix is assigned to the interface (by `dhcp6leased`), `rad` will advertise it.* Recall above we configured `dhcp6leased` to assign the first /64 to `ix0`.

This is consistent with OpenBSD's philosophy of minimal config when possible, and it makes IPv6 Just Work™

## 7. Enable and start `rad`
Wait until dhcp6leased has received the delegated prefix. Then, enable and start rad, so that it advertises the correct prefix and DNS info on LAN.  
```sh
rcctl enable rad
rcctl start rad
```
## 8. Send GUA to `ix0`:  (IPv6)

Recall from above that we configured `hostname.ix0` and included the line: 

```conf
inet6
```
This directs `slaacd` to run on the LAN interface. Therefore, after receiving router advertisements from `rad`, `slaacd` will assign a GUA to `ix0` derived from the prefix within the file `/var/db/dhcp6leased/ix1`

```sh
rcctl enable slaacd
rcctl start slaacd
```

## 8b. Enable and start `dhcpd` to serve IPv4 addresses on the LAN:
```sh
rcctl enable dhcpd
rcctl set dhcpd flags ix0
rcctl start dhcpd
```
## 9. Reboot and verify.
Rebooting is not strictly necessary at this point, but it will prove our configurations survive a system restart.

```sh
reboot
```

Verify LAN address assignments:  (IPv4/IPv6)

```sh
ifconfig ix0
```

In addition to your IPv4 LAN address, you should see your IPv6 Global Unicast Address (GUA):

```
...
inet 192.168.1.1
...
inet6 2600:4040:AAAA:BBBB::1 prefixlen 64
```

*You are getting close to the summit!*

## The IPv6 Prefix Delegation model (This ain't IPv4!):

Verizon FiOS assigns a **delegated IPv6 prefix** (typically a /56) to your router via DHCPv6 rather than assigning a Global Unicast Address (GUA) directly to the router’s WAN interface. This aligns with IPv6’s design principles, which support end-to-end connectivity without the need for NAT.

## How It Works on OpenBSD 7.7

**`dhcp6leased`** handles all DHCPv6 communication with Verizon on the WAN interface. It sends a request for prefix delegation (IA_PD) and receives a delegated prefix, usually a /56. It then subdivides that prefix according to its configuration and **records subprefixes assigned to each LAN interface** in its internal lease database located at `/var/db/dhcp6leased/`.

Note: `dhcp6leased` does **not directly configure addresses** on interfaces- it only manages prefix delegation and records the mapping for other daemons to use.

**`rad`** (Router Advertisement Daemon) reads the assigned subprefixes from the `dhcp6leased` lease database and sends Router Advertisements (RAs) on the LAN interface(s), to advertise the corresponding subnet (and DNS, if configured as such). This allows clients to self-configure IPv6 addresses using SLAAC.

> ⚠️ `rad` must be restarted or reloaded manually to pick up new prefix data.

**`slaacd`** runs on the OpenBSD router's LAN interface(s) (LAN client devices also use ther own **SLAAC**). `slaacd` configures IPv6 addresses using SLAAC, and installs a default route via the router’s link-local address (fe80::/10).

### Why This Design Works

**No WAN GUA Needed**  
Unlike IPv4, where a public WAN address is necessary for NAT, IPv6 routers simply route packets using their delegated prefix. There’s no need for a GUA on the WAN interface in this setup.

**Link-Local Sufficient**  
The router's WAN interface uses a link-local IPv6 address (`fe80::/10`) to communicate with Verizon’s upstream router, which is sufficient for both routing and DHCPv6.

**Delegated GUA on LAN**  
The router receives its own IPv6 address on each LAN interface by processing its own Router Advertisements via `slaacd`. These addresses are derived from the delegated prefix.

**Efficient and Compliant**  
This design reflects IPv6 best practices and conserves address space while enabling native, end-to-end IPv6 routing for all LAN clients- without NAT.

## 10. DNS and `unbound`  (IPv4/IPv6)

Unbound is a recursive, caching DNS resolver with DNSSEC validation, DNS over TLS, and RPZ support. The following example allows for using the root servers or forwarding DNS over TLS to Google, as well as blocking malicious domains, depending on how you wish to proceed.

### `/var/unbound/etc/unbound.conf`

```conf
# /var/unbound/etc/unbound.conf
# uncomment what is needed/preferred
server:
    interface: ix0  # All IPv4 and IPv6 addresses assigned to this interface. (192.168.1.1, GUA and ULA)
    interface: 127.0.0.1  # Loopback on the router itself
    interface: ::1  # ipv6 Loopback on the router itself
    
    private-address: 192.168.0.0/16
    private-address: fd00::/8
    private-address: fe80::/10
    private-domain: home.arpa

    tls-cert-bundle: "/etc/ssl/cert.pem"
    # Include a localhosts.conf file for reverse lookups
    # see https://openbsdrouterguide.net/
    #include: "/var/unbound/etc/unbound.localhosts.conf"

    local-zone: "192.168.1.0/24" static

    # Default deny everything:
    access-control: 0.0.0.0/0 refuse
    access-control: ::0/0 refuse
    # Allow:
    access-control: 127.0.0.0/8 allow
    access-control: ::1 allow
    access-control: 192.168.1.0/24 allow
    access-control: fd00:AAAA:BBBB:CCCC::/64 allow  # <--- Replace with your actual ULA.

    hide-identity: yes
    hide-version: yes
    prefetch: yes

    # Enable DNSSEC:
    # auto-trust-anchor-file: "/var/unbound/db/root.key"

    # Uncomment to load rpz module and configure below:
    # module-config: "respip validator iterator"

    cache-min-ttl: 3600
    serve-expired: yes

remote-control:
    control-enable: yes
    control-interface: /var/run/unbound.sock
# Uncomment to forward to Google DNS
#forward-zone:
    #name: "."
    #forward-tls-upstream: yes
    #forward-addr: 8.8.8.8@853#dns.google
    #forward-addr: 8.8.4.4@853#dns.google
    #forward-addr: 2001:4860:4860::8888@853#dns.google
    #forward-addr: 2001:4860:4860::8844@853#dns.google

#rpz:
#    name: my preferred blocklist
#    url: https://raw-rpz-blocklist  # enter the url of a well-maintained, raw blocklist in rpz format
#    rpz-action-override: nxdomain
```
Notice the `interface: ix0` clause.

This directs `unbound` to **listen** on all addresses assigned to the LAN interface. 

In our example, this now includes:
- 192.168.1.1
- Our **ULA** fd00:AAAA:BBBB:CCCC::1
- Our **GUA** 2600:4040:AAAA:BBBB:CCCC::1

However, notice the `access-control:` section:
```conf
access-control: 192.168.1.0/24 allow
access-control: fd00:AAAA:BBBB:CCCC::/64
```
Only the RFC 1918 subnet 192.168.1.0/24 subnet and our ULA subnet fd00:AAAA:BBBB:CCCC::/64 are granted access.
So, essentially, unbound listens on the interface, and the `access-control:` limits which addresses are granted access. Therefore, the `interface: ix0` is being used as shorthand. We could also explicitly configure by addresses using `interface:` if we choose.

Enable and start `unbound`
```sh
rcctl enable unbound
rcctl start unbound
```

## 11. Reboot and test
Ensure dhcpleased, dhcpd, dhcp6leased, rad, slaacd and unbound are enabled and running:
```sh
rcctl ls on
rcctl ls started
```
```sh
reboot
```
Test your configuration from the router using the tools of your choice, e.g.:
```sh
ifconfig ix0
ping6 example.com
dig -6 @fd00:AAAA:BBBB:CCCC::1 example.com AAAA
dig example.com AAAA
```
https://test-ipv6.com/ can be utilized from clients.

# What is happening here (summary):
| Step | Daemon               | Role                                                                                                       |
| ---- | -------------------- | ---------------------------------------------------------------------------------------------------------- |
| 1    | **`dhcp6leased`**    | Sends DHCPv6 request to Verizon on WAN `ix1`, receives delegated prefix, writes lease info to `/var/db/dhcp6leased/ix1` |
| 2    | **`rad`**            | Reads prefix info from `dhcp6leased`, advertises delegated subprefix, gateway (and DNS) on LAN `ix0`             |
| 3    | **`slaacd`**         | Runs on LAN clients and router LAN interface; processes RAs, generates and configures GUA and default route  |
| 4    | **`unbound`**        | Serves DNS to LAN clients using ULA address                                                               |
| 5    | **`dhcpleased`**     | Handles IPv4 DHCP on WAN `ix1`, assigns IPv4 address and default route                                    |
| 6    | **`dhcpd`**          | The IPv4 dhcp server; hands out IPv4 local addresses on LAN |

That's it! Hopefully, this guide has been valuable to you. 

### Troubleshooting Table 
The following table may be useful, especially if you configured your system to be very close to the examples.
| Problem | Symptom | Check/Command | Possible Cause(s) |
| :--- | :--- | :--- | :--- |
| **No Internet (IPv4)** | No IPv4 address on `ix1` (`ifconfig ix1`). LAN clients get no IPv4 address. | `rcctl ls on` <br> `rcctl ls started` <br> `rcctl restart dhcpleased` <br> `ifconfig ix1` | <ul><li>`dhcpleased` is not running/enabled.</li><li>`dhcpd` is not running/enabled.</li><li>`pf.conf` is blocking DHCP traffic (ports 67/68 UDP).</li></ul> |
| **No Internet (IPv6)** | No IPv6 GUA on `ix0` (`ifconfig ix0`). LAN clients have no IPv6 address. `ping6 ipv6.google.com` fails. | `rcctl ls on` <br> `rcctl ls started` <br> `dhcp6leased -d` <br> `ifconfig ix0` | <ul><li>`dhcp6leased`, `rad`, or `slaacd` is not running/enabled.</li><li>`dhcp6leased` did not get a prefix from Verizon.</li><li>`rad.conf` is misconfigured (no prefix or DNS).</li><li>`pf.conf` is blocking `ipv6-icmp` or DHCPv6 traffic (ports 546/547 UDP).</li><li>`rad` needs to be restarted after a new prefix is acquired.</li></ul> |
| **DNS Resolution Fails** | `dig example.com` fails from router. Clients cannot resolve names. | `rcctl ls on` <br> `rcctl ls started` <br> `unbound-checkconf` <br> `unbound-control status` <br> `cat /etc/resolv.conf` | <ul><li>`unbound` is not running/enabled.</li><li>`unbound.conf` has syntax errors.</li><li>`unbound.conf` `access-control` is **too restrictive**.</li><li>`resolv.conf` on router is not pointing to 127.0.0.1 and ::1.</li><li>Firewall is blocking port 53.</li></ul> |
| **Clients get IPv4 but not IPv6** | `ifconfig ix0` shows a GUA, but clients only get an IPv4 address. | `rcctl ls on` <br> `rcctl ls started` <br> `tcpdump -ni ix0 ip6` <br> `rcctl restart rad` <br>  | <ul><li>`rad` is not running or enabled.</li><li>Firewall is blocking IPv6 traffic.</li><li>`rad.conf` is missing or misconfigured.</li><li>`rad` may need to be restarted to pick up the new prefix.</li><li>Client OS/configuration issue.</li></ul> |

## **ENJOY!**

If you have not done so already, I recommend setting up `dhcpd.conf` and `unbound.conf` for local hostname resolution by using static reservations. An excellent guide which expands on this is here: https://openbsdrouterguide.net/

## License

OpenBSD and this project are licensed under the **BSD style License** in the source directory of this repository.

## 👤 Author

**Misfit-138**

**"Standing on the shoulders of giants"**

Without the following resources and people, this guide would have been impossible for me.

* Florian Obser: Created OpenBSD's modern IPv6 infrastructure by writing slaacd (OpenBSD 6.2), rad (OpenBSD 6.4), and dhcp6leased (OpenBSD 7.6) - the three daemons that handle IPv6 autoconfiguration, router advertisements, and prefix delegation respectively. https://www.youtube.com/watch?v=Q4b26mqb_G8
* N.M. Hansteen: Author of **The Book of PF, 4th edition** (Yes, I preordered and got the early access PDF!)
* Jeremy Evans: https://code.jeremyevans.net/2024-11-03-adding-ipv6-to-my-home-network.html
* Jeffrey Forman: https://write.jeffreyforman.net/posts/2022/verizon-fios-native-ipv6/
* https://www.openbsd.org/faq/pf/example1.html
* https://openbsdrouterguide.net/
* https://dataswamp.org/



## ❓ FAQ

### Why OpenBSD?

Because it does the job simply, securely, and elegantly. It's ideal for a firewall/router role (I use it on my desktop and laptop as well). OpenBSD is simply the most beautiful modern OS.
I found it 20 years ago, (though I am still a novice) fell in love with it, and I have donated to the project. I hope this page contributes to the project by helping someone.

### Why not Linux or FreeBSD?

They're great tools, but I find them unnecessarily complex, especially for routing/firewalling. OpenBSD has fewer surprises and a consistent, coherent philosophy.

### Why do you include Google DNS?

- In my region, Google DNS is fast and reliable.
- Other providers and root servers did not perform well for me.
- Feel free to configure your system to your own preference.
  
## 🙏 Final Thoughts

This guide exists to help others get a solid OpenBSD dual stack router working with Verizon FiOS. I spent weeks testing, reading, and tweaking. If it helps one person, it was worth the effort. If you are that person, or, if you find value in this guide, I would be honored if you clicked on the star. Thanks for reading.
