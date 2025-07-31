# Demystifying OpenBSD 7.7 Router/Firewalls with Verizon FiOS Dual Stack (IPv4 + IPv6)

![Example Image](openbsdmountain2.jpg)

## Overview

In addition to explaining the basic functionality of the OpenBSD IPv6 system, this guide describes a **proven method** to configure an OpenBSD 7.7-based router/firewall with **dual stack IPv4 and IPv6** using residential **Verizon FiOS**. It includes support for dynamic IPv6 prefix delegation, DNS advertisement to LAN clients, DNSSEC, root server querying, or DNS over TLS using `unbound` to forward to Google, and optionally, DNS blocklisting utilizing RPZ. 
## Why?

- Because IPv6 is new and mysterious to me, and in my pursuit of understanding it, I wanted to share something with the community that may be helpful.

- Because simplicity is beautiful; OpenBSD is beautiful. It excels as a firewall/router platform, offering correctness, security, elegance and transparency. It resists the trend to incorporate bloatware disguised as features, inefficiency disguised as modernity, and unnecessary complexity. This guide shares a working configuration to help others build a reliable OpenBSD-based network gateway using the stable, default system tools found in the `base` system.

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

### 1. Install OpenBSD 7.7
Official instructions at: https://www.openbsd.org/faq/faq4.html

During the install set up a WAN interface configured with automatic IPv4/IPv6 and a LAN interface with static IPv4 for now.

### 2. Check for *running* and *enabled* daemons 

```sh
rcctl ls started
rcctl ls on
```
`slaacd` **should** be enabled by default in a fresh install, but disable for now:
```sh
rcctl stop slaacd
rcctl disable slaacd
```

### 3. Disable `resolvd` (recommended)  
If you want full control over DNS (to avoid using ISP DNS):

```sh
rcctl stop resolvd
rcctl disable resolvd
```
## 4. Edit configuration files
### `/etc/sysctl.conf`
Turning the system into a router that will forward IPv4/IPv6 packets between network interfaces is simple and easy:
```conf
# sysctl.conf for router/firewall
net.inet.ip.forwarding=1
net.inet6.ip6.forwarding=1
```
### `/etc/dhcpd.conf`
A simple, sane, working example:
```conf

# /etc/dhcpd.conf
subnet 192.168.1.0 netmask 255.255.255.0 {
	option routers 192.168.1.1;
	domain-name-servers 192.168.1.1;
	range 192.168.1.10 192.168.1.254;
}
```
### `/etc/resolv.conf`
This file configures the *router's* resolving behavior.
```conf
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

`search home.arpa`→ If you type just a short hostname (like, myserver), the system will try appending .home.arpa to it, making it myserver.home.arpa. This helps with resolving names in your home network automatically.


### Create `/etc/dhcpleased.conf` 

dhcpleased does the following:
- Sends DHCP Discover/Request messages to find and lease an IPv4 address.
- Receives offers from a DHCP server, selects one, and requests it.
- Writes lease info to /var/db/dhcpleased/ so it persists across reboots.
- Configures your interface with:
  - IPv4 address
  - Default gateway (route add default)
  - DNS resolvers (written to /etc/resolv.conf)
  - Renews leases as needed to keep the IP assignment active.
The following simple example with ignore ISP DNS assignment (recommended in this guide). Along with disbling `resolvd`, (recommended above) this give us full control over our DNS:
```conf
interface ix1 { ignore dns }
```
`dhcpleased` is automatically started by rc.d on interfaces marked with `inet autoconf` in `/etc/hostname.if`, so it should be enable and running. 

Restart with:
```sh
rcctl restart dhcpleased
```
At this point, you may wish to go back and re-check your resolv.conf, to ensure it has not been overwritten.

### `/etc/dhcp6leased.conf`:
This simple file is all that is needed, and is quite self explanatory:
```conf
# dhcp6leased.conf for OpenBSD 7.7
# WAN interface: ix1, LAN interface: ix0

# Request prefix delegation and assign first /64 to LAN
request prefix delegation on ix1 for { ix0/64 }

```
### `/etc/hostname.ix0` (LAN):

### Create a ULA (Unique Local Address)- IPv6's equivalent to RFC 1918 private addresses like 192.168.x.x in IPv4.
Why it's useful:

* ULAs provide stable, predictable addresses for local network communication.
* Unlike global IPv6 addresses (which can change when your ISP changes your delegated prefix), ULAs remain constant.
* Ensures local services and device-to-device communication continues working even if your ISP prefix changes-Provides a fallback for local network services.

You can easily create your own random ULA using `jot`:
```sh
jot -r 6 0 255 | xargs printf "fd00:%02x%02x:%02x%02x:%02x%02x::1/64\n"
```
The output should be something like:
```sh
fd00:AAAA:BBBB:CCCC::1/64
```
We'll use this ULA as an alias for the LAN interface in `hostname.if` and to plug into `rad.conf` and `unbound.conf`.
```sh
# /etc/hostname.ix0 (LAN):
inet 192.168.1.1 255.255.255.0 192.168.1.255
inet6
inet6 alias fd00:AAAA:BBBB:CCCC::1/64  # ULA alias for LAN interface (Create your own.)
```

### `/etc/hostname.ix1` (WAN):
Simple, clean and brainless. And, it *just works*:
```sh
inet autoconf
inet6 autoconf
```
Enable and start `dhcpd` to serve IPv4 addresses on the LAN:
```sh
rcctl enable dhcpd
rcctl set dhcpd flags ix0
rcctl start dhcpd
```
## 🔥 `pf.conf` (Firewall Rules)

A clean and concise dual stack PF configuration with minimal logging, which works with both IPv4 and IPv6. It is based on a "block all in, let anything out" foundation, with some extra security against spoofing, and selected filtering for functionality; this is generally fine for a trusted home LAN, but again, KNOW WHAT YOU ARE DOING.

## *IPv6 essential considerations:*
- There must be a route-to or pass out rule in `pf.conf` for IPv6 outbound from LAN to WAN. This is easy to forget and will silently block IPv6. Our example let's everything out- `pass out quick inet6 keep state`.
- ICMPv6 needs to be allowed on WAN and LAN or many things break — neighbor discovery, path MTU discovery, RA, etc. This is covered by `pass in quick inet6 proto ipv6-icmp from any to any icmp6-type {
    echoreq, echorep, unreach, toobig, timex, paramprob,
    neighbrsol, neighbradv, routersol, routeradv
} keep state`
- Inbound DHCPv6 client <-> server traffic (udp port 546 <- 547) must be enabled on WAN ix1. Our example covers this with `pass in quick on egress inet6 proto udp from any port 547 to any port 546`.

# ⚠️ IMPORTANT SECURITY DISCLAIMER ⚠️

**THIS CODE DEFINES THE SYSTEM FIREWALL BEHAVIOR. USE AT YOUR OWN RISK.**

```pf
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
block in quick on egress inet6 from fd00::/8 to any 
#block in quick on egress inet6 from 2600:4040:AAAA:BBBB::/64 to any  # uncomment after IPv6 global address is acquired and insert your actual address

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

## 📡 `rad.conf` (Router Advertisement)

### Initial Configuration

#### Create `/etc/rad.conf`
We'll create a simple `rad.conf`, only as a logical placeholder for now.
```conf
interface ix0 { }
```
This simply directs `rad` to the current LAN interface (`ix0`) we are using in our example.

*Do not enable rad at this point.* 

## 5a. Acquire IPv4 address and routes and forward packets.

#### Start (or restart) `dhcpd`, `dhcpleased` for IPv4 if not already running:
```sh
rcctl ls on
rcctl ls started
```
## 5b. Acquire Delegated Prefix (IPv6)

#### Enable and run `dhcp6leased` manually to observe prefix delegation

```sh
rcctl enable dhcp6leased
dhcp6leased -d
```

You should see something like:

```
...prefix delegation #1 2600:4040:AAAA:BBBB::/56 received on ix1 from server ...
```
Copy this prefix; We will use it to create your GUA, for the antispoofing rule in pf.conf.
#### 6. Send GUA to `ix0`:
Stop `dhcp6leased` (Ctrl+C) and start it normally:
```sh
rcctl start dhcp6leased
```
Start `slaacd` to jumpstart assigning the GUA to ix0:
```sh
rcctl enable slaacd
rcctl start slaacd
```
#### 7. Update `rad.conf` with your ULA (from hostname.ix0) to advertise to your LAN:

```conf
interface ix0 {
    dns {
        nameserver fd00:AAAA:BBBB:CCCC::1
    }
}
```
*Wait until dhcp6leased has received the delegated prefix (you can monitor with `ifconfig ix0`), then, enable and start `rad`. This ensures Router Advertisements carry the correct prefix and DNS information.*
#### 8. Enable and start `rad`

```sh
rcctl enable rad
rcctl start rad
```

#### 9. Verify interface address by checking your LAN:

```sh
ifconfig ix0
```

In addition to your IPv4 address, you should see your IPv6 Global Unicast Address (GUA):

```
inet6 2600:4040:AAAA:BBBB::1 prefixlen 64
```

*You are getting close to the summit!*

## The IPv6 Prefix Delegation model (This ain't IPv4!):

Verizon FiOS assigns a **delegated IPv6 prefix** (typically a /56) to your router via DHCPv6 rather than assigning a Global Unicast Address (GUA) directly to the router’s WAN interface. This aligns with IPv6’s design principles, which support end-to-end connectivity without the need for NAT.

### How It Works on OpenBSD 7.7

**`dhcp6leased`** handles all DHCPv6 communication with Verizon on the WAN interface. It sends a request for prefix delegation (IA_PD) and receives a delegated prefix, usually a /56. It then subdivides that prefix according to its configuration and **records subprefixes assigned to each LAN interface** in its internal lease database located at `/var/db/dhcp6leased/`.

Note: `dhcp6leased` does **not directly configure addresses** on interfaces—it only manages prefix delegation and records the mapping for other daemons to use.

**`rad`** (Router Advertisement Daemon) reads the assigned subprefixes from the `dhcp6leased` lease database and sends Router Advertisements (RAs) on the LAN interface(s), to advertise the corresponding subnet (and DNS, if configured as such). This allows clients to self-configure IPv6 addresses using SLAAC.

> ⚠️ `rad` must be restarted or reloaded manually to pick up new prefix data.

**`slaacd`** runs on the OpenBSD router's LAN interface(s) (as well as LAN client devices). On all of these, it listens for Router Advertisements sent by the OpenBSD router itself, configures IPv6 addresses using SLAAC, and installs a default route via the router’s link-local address (fe80::/10).

### Why This Design Works

**No WAN GUA Needed**  
Unlike IPv4, where a public WAN address is necessary for NAT, IPv6 routers simply route packets using their delegated prefix. There’s no need for a GUA on the WAN interface in this setup.

**Link-Local Sufficient**  
The router's WAN interface uses a link-local IPv6 address (`fe80::/10`) to communicate with Verizon’s upstream router, which is sufficient for both routing and DHCPv6.

**Delegated GUA on LAN**  
The router receives its own IPv6 address on each LAN interface by processing its own Router Advertisements via `slaacd`. These addresses are derived from the delegated prefix.

**Efficient and Compliant**  
This design reflects IPv6 best practices and conserves address space while enabling native, end-to-end IPv6 routing for all LAN clients—without NAT.


#### 10. Update `pf.conf` IPv6 antispoofing rule

Uncomment and update this line with your GUA:

```pf
block in quick on egress inet6 from 2600:4040:AAAA:BBBB::/64 to any
```
This rule blocks IPv6 packets coming from the internet that claim to be from your GUA (2600:4040:AAAA:BBBB::/64). It is essentially serving as an IPv6 anti-spoofing measure.
Right below it you will see `block in quick from fd00::/8` These are private-use addresses and should never appear from outside. Blocking them explicitly is good practice.

Why this matters:

Legitimate traffic from your internal network should never arrive through your external interface - it should only come from inside your network. If packets with your internal addresses show up on the external interface, they're fake (spoofed).

So, in addition to  `antispoof quick for { egress $lan }` we are layering security.

`antispoof` handles general interface/source mismatches; it blocks packets that come from the wrong interface (e.g., a LAN IP on the WAN), while the `block in quick on egress inet6 from fd00::/8 to any` and `block in quick on egress inet6 from $gua_prefix to any` offer explicit, guaranteed protection based on address — not just interface.

So:

`antispoof` checks where a packet came from.

The explicit IPv6 rules check what kind of address a packet is using.

Both matter. Together, they catch different kinds of spoofing.

#### 11. Reload `pf` rules

```sh
pfctl -f /etc/pf.conf
```

## 12. DNS and `unbound`

Unbound is a recursive, caching DNS resolver with DNSSEC validation, DNS over TLS, and RPZ support. The following example allows for using the root servers or forwarding DNS over TLS to Google, as well as blocking malicious domains, depending on how you wish to proceed.

### `/var/unbound/etc/unbound.conf`
Notice `interface-automatic: yes` is enabled so Unbound binds to GUA dynamically if assigned late.
```conf
# unbound.conf
# uncomment what is needed/preferred
server:
    interface: 127.0.0.1
    interface: ::1
    interface: fd00:AAAA:BBBB:CCCC::1  # replace with your ULA from hostname.ix0
    interface-automatic: yes
    do-ip6: yes

    private-address: 192.168.0.0/16
    private-address: fd00::/8
    private-address: fe80::/10
    private-domain: home.arpa.

    tls-cert-bundle: "/etc/ssl/cert.pem"
    # Include a localhosts.conf file for reverse lookups
    # see https://openbsdrouterguide.net/
    #include: "/var/unbound/etc/unbound.localhosts.conf"

    local-zone: "192.168.1.0/24" static

    access-control: 0.0.0.0/0 refuse
    access-control: ::0/0 refuse
    access-control: 127.0.0.0/8 allow
    access-control: ::1 allow
    access-control: 192.168.1.0/24 allow
    access-control: fd00:AAAA:BBBB:CCCC::/64 allow

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
#    url: https://raw-rpz-blocklist
#    rpz-action-override: nxdomain
```
Enable `unbound`
```sh
rcctl enable unbound
```

## 13. Reboot and test
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



## **ENJOY!**

If you have not done so already, I recommend setting up `dhcpd.conf` and `unbound.conf` for local hostname resolution by using static reservations, in conjunction with unbound.conf. An excellent guide which expands on this is here: https://openbsdrouterguide.net/

## License

OpenBSD and this project are licensed under the **BSD style License** in the source directory of this repository.

## 👤 Author

**Misfit-138**

**"Standing on the shoulders of giants"**

Without the following resources and people, this guide would have been impossible for me.

* Florian Obser: Created OpenBSD's modern IPv6 infrastructure by writing slaacd (OpenBSD 6.2), rad (OpenBSD 6.4), and dhcp6leased (OpenBSD 7.6) - the three daemons that handle IPv6 autoconfiguration, router advertisements, and prefix delegation respectively.
* Jeremy Evans: https://code.jeremyevans.net/2024-11-03-adding-ipv6-to-my-home-network.html
* Jeffrey Forman: https://write.jeffreyforman.net/posts/2022/verizon-fios-native-ipv6/
* https://www.openbsd.org/faq/pf/example1.html
* https://openbsdrouterguide.net/
* https://dataswamp.org/



## ❓ FAQ

### Why OpenBSD?

Because it does the job simply, securely, and elegantly. It's ideal for a firewall/router role (I use it on my desktop and laptop). OpenBSD is simply the most beautiful modern OS.
I found it 20 years ago, (though I am still a novice) fell in love with it, and I have donated to the project. I hope this page contributes to the project by helping someone.

### Why not Linux or FreeBSD?

They're great tools, but I find them unnecessarily complex, especially for routing/firewalling. OpenBSD has fewer surprises and a consistent, coherent philosophy.

### Why Google DNS?

- In my region, Google DNS is fast and reliable.
- Other providers and root servers did not perform well for me.
- Feel free to configure your system to your own preference.
  
### Aren’t Verizon FiOS delegated prefixes dynamic?

Theoretically, yes; They can change. If this concerns you, I recommend creating a script which detects the change, writing it to a file, and `include` the file in `pf.conf`, so the IPv6 antispoof rule remains intact.

## 🙏 Final Thoughts

This guide exists to help others get a solid OpenBSD dual stack router working with Verizon FiOS. I spent weeks testing, reading, and tweaking. If it helps one person, it was worth the effort.
