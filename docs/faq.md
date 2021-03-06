# F.A.Q.

- [Why is this project called dlan? A LAN is _local_ by definition, how can it be distributed?](#why-is-this-project-called-dlan-a-lan-is-local-by-definition-how-can-it-be-distributed)
- [How does mesh networking work?](#how-does-mesh-networking-work)
    * [Can't we skip the VXLAN encapsulation, and use wireguard directly?](#cant-we-skip-the-vxlan-encapsulation-and-use-wireguard-directly)
    * [So if multicast worked over wireguard, it'd be fine?](#so-if-multicast-worked-over-wireguard-itd-be-fine)
    * [But can't i theoretically configure wireguard so that AllowedIPs enumerates all subnets that are reachable by that peer?](#but-cant-i-theoretically-configure-wireguard-so-that-allowedips-enumerates-all-subnets-that-are-reachable-by-that-peer)
    * [Why not use L2TPv3 instead of VXLAN?](#why-not-use-l2tpv3-instead-of-vxlan)
    * [Which VXLAN IDs are used among hosts?](#which-vxlan-ids-are-used-among-hosts)
    * [Does OSPF advertise also the WAN network?](#does-ospf-advertise-also-the-wan-network)
    * [Can two routers live in the same home LAN?](#can-two-routers-live-in-the-same-home-lan)
- [I see that the wireguard IP is unique in the mesh; can't we just use those and avoid configuring additional loopback IPs on the nodes?](#i-see-that-the-wireguard-ip-is-unique-in-the-mesh-cant-we-just-use-those-and-avoid-configuring-additional-loopback-ips-on-the-nodes)
    * [Why is there a conflict?](#why-is-there-a-conflict)
- [Can i have a static DNS entry pointing to multiple IPs?](#can-i-have-a-static-dns-entry-pointing-to-multiple-ips)
- [Regarding IPv6 support...](#regarding-ipv6-support)
    * [Why use dnsmasq-dhcpv6? Isn't odhcpd enough?](#why-use-dnsmasq-dhcpv6-isnt-odhcpd-enough)
    * [Is DHCPv6-PD used?](#is-dhcpv6-pd-used)
    * [Can i give downstream interfaces global IPv6 ranges?](#can-i-give-downstream-interfaces-global-ipv6-ranges)
- [Regarding nginx configuration...](#regarding-nginx-configuration)
    * [Why not use NAT table rules instead of nginx, for exposing services? Or why not HAproxy?](#why-not-use-nat-table-rules-instead-of-nginx-for-exposing-services-or-why-not-haproxy)
    * [Can i use a DNS record that changes over time in a `dlan_proxy` rule?](#can-i-use-a-dns-record-that-changes-over-time-in-a-dlan_proxy-rule)
    * [Can i use a DNS record that points to multiple IPs in a `dlan_proxy` rule?](#can-i-use-a-dns-record-that-points-to-multiple-ips-in-a-dlan_proxy-rule)
    * [How to listen on both IPv4 and IPv6 addresses, for a single `dlan_proxy` entry?](#how-to-listen-on-both-ipv4-and-ipv6-addresses-for-a-single-dlan_proxy-entry)
- [Troubleshooting tips? Known issues? Gotchas?](#troubleshooting-tips-known-issues-gotchas)


### Why is this project called dlan? A LAN is _local_ by definition, how can it be distributed?

It's true that the concept of a geographically distributed LAN doesn't make sense (that'd be a WAN technically). The goal of the project is not to create one: it's to replicate the ease of configuration of a home LAN in a geographically distributed infrastructure.

E.g. in a LAN we expect devices to be able ping each other, and the possibility to configure port forwardings to expose some services. Dlan tries to achieve this same usability, but without being constrained by device location.

You could see it as "splitting your home router across locations". E.g. you run your own hardware in your home's network (attached to dlan node `router1`, which is cascaded to your NATting home router and thus doesn't have a public IP), and expose that service through the public IP of a separate, publicly reachable machine (e.g. `router2`, a machine running on OVH). `router1` and `router2` act as a "distributed home router".


### How does mesh networking work?

Between each peered pair of routers (so two routers that mention each other in their `dlan_links` parameter), there's a wireguard+VXLAN tunnel. So packets that travels between two adjacent routers are in fact wireguard packets. Digging deeper into the packet, we can identify the following encapsulation layers:
- The wireguard packet. Src/dest IPs are the WAN-side IPs of the two routers. This wireguard packet's payload is...
- ...the VXLAN packet. Src/dest IPs are the two router's wireguard interfaces' private IPs. This VXLAN packet's payload is...
- ...the IP payload itself. Src/dest IPs here are the actual source/destination nodes' IPs (the source address' value varies between IP protocols, but any private one works, thanks to OSPF).

Do note that the IP of the vxlan interfaces themselves only appear in the kernel routing table, written by the bird daemon based on the OSPF mesh informations, and is mainly used for next-hop MAC lookups when routing packets across the internal network).

OSPF is used to distribute all subnets a certain router manages on LAN side (plus its loopback address) so that those are reachable by any other router in the mesh.

##### Can't we skip the VXLAN encapsulation, and use wireguard directly?
Unfortunately not: OSPF, which is needed to build a resilient IP mesh, needs an L2 link to run as it relies on multicast traffic. Due to the additional peer mapping logic (the AllowedIPs directive is looked up, do decide which peer gets a package), multicast can't work over wireguard, and thus we need to create an L2 tunnel.

##### So if multicast worked over wireguard, it'd be fine?
 No, even if OSPF worked there'd be another more issue: due to the same peer mapping logic, the source and destination IPs of packets travelling through the wireguard interfaces MUST be those of the peers of the link (unless you configure AllowedIPs for multi-hop scenarios, see next point). VLXAN does the trick here, as no matter the src/dest IPs of your actual IP packets, as soon as they enter a vxlan interface they will get encapsulated in a vxlan packet that has the src/dest addresses of the src/dest wireguard interfaces of the routers in this hop (only adjacent routers in the mesh are vxlan peers, so it's always a single-hop for any vxlan-encapsulated frame).

##### But can't i theoretically configure wireguard so that AllowedIPs enumerates all subnets that are reachable by that peer?
Theoretically yes, but to have this configuration be autonomous you still need an IGP protocol to monitoring reachability of peered routers, and the bird daemon doesn't support writing the wireguard config. And even if it did, Wireguard only accepts a certain route behind a single peer (due to the peer mapping logic), which is not what is expected from an IGP protocol (where a single route may be reachable through different routers, achieving ECMP).

##### Why not use L2TPv3 instead of VXLAN?
Mainly due to poor OpenWrt support. It would have been a valid alternative to VXLAN.

##### Which VXLAN IDs are used among hosts?
A value is autogenerated per-host-pair number, using `dlan_maxhosts`, so that it's guaranteed to be unique in the whole mesh.

##### Does OSPF advertise also the WAN network?
No, that's not supported. The dlan network doesn't care about what's beyond the WAN port, there are only clients and dragons there. It's also impossible to guarantee that the network behind it is unique, as it's not under the router control and e.g. most home network use ranges `192.168.0.0/24` or `192.168.1.0/24`.

##### Can two routers live in the same home LAN?
Yes, they can. You can peer them to each other by using their private IPs, so that traffic between the two will travel through the home LAN (still encrypted). If you do in non-trivial installations, it's recommended to also tune the link weights, to ensure traffic between the local routers effectively travels only through the local link.

Do remember that in all cases, dlan routers may talk to each other only through their WAN interface. Any other combination (e.g. LAN port of a router connected to another router's LAN, or to another router's WAN) is not supported. Only client machines (that is, the machines running your services in the internal network) should be connected to the LAN ports.


### I see that the wireguard IP is unique in the mesh; can't we just use those and avoid configuring additional loopback IPs on the nodes?

No, we can't unfortunately. The loopback address should be advertised through OSPF for global reachability within the mesh, and we can't do it with the wireguard IP: if the wireguard IP is advertised through OSPF to the other routers, it would break the connectivity with all of its adjacent routers, due to conflicting routes.

##### Why is there a conflict?
Each node already has `/32` routes to its wireguard peers (= adjacent routers), before OSPF even runs; packets destined to a certain wireguard peer MUST enter the wireguard interface (where the kernel module itself does the demultiplexing, looking at the AllowedIPs of all peers). Then bird starts, and begins receiving route infos through OSPF: for each known subnet, it writes an entry in the kernel routing table, so that packets between routers will travel through the same link where OSPF protocol runs, and that means the VXLAN interface in dlan's case. This means: if OSPF were to advertise the wireguard IPs, then it would write in the routing table that packets destined to wireguard IPs of adjacent routers should be sent THROUGH the vxlan interface...which conflicts with the initial assumption that wireguard packets must enter the wireguard interface. As the wireguard IPs are used as the vxlan interfaces' endpoint IPs, the vxlan tunnels will of course also not work either, and connectivity is broken.


### Can i have a static DNS entry pointing to multiple IPs?

Yes, simply create separate entries under `dlan_host_records` that have the same name but different IPs.


### Regarding IPv6 support...

##### Why use dnsmasq-dhcpv6? Isn't odhcpd enough?
odhcpd seems to do its job just fine as an IPv6 DHCP server. The issue here is the separation between dnsmasq and odhcpd, which may hold configuration surprises we'd rather avoid.

For instance: if both are used (as DHCP servers; dnsmasq for IPv4 and odhcpd for IPv6), the way OpenWrt integrates them to serve IPv6 records (as dnsmasq knows nothing about those leases) is to have odhcpd write its leases to a file, and dnsmasq pick those up through that file with the `addn-hosts` flag. However, I hit an out-of-the-box bug in 19.07.2 in which dnsmasq couldn't read the file written by odhcpd due to permission issues. Using dnsmasq also for DHCPv6 we remove this additional possible failure point.

##### Is DHCPv6-PD used?
No, IPv6 support is limited to static configuration of prefixes for downstream interfaces.

##### Can i give downstream interfaces global IPv6 ranges?
No, only private network addresses are supported (E.g. ULA). nginx will always stand between clients and your services.


### Regarding nginx configuration...

##### Why not use NAT table rules instead of nginx, for exposing services? Or why not HAproxy?
The NAT table rules limit the number of upstream IPs to 1, which is not ideal asd you may want to expose your service through multiple servers (e.g. think kubernetes NodePort services). HAproxy does not support UDP loadbalacing.

##### Can i use a DNS record that changes over time in a `dlan_proxy` rule?
You can technically, but there is a catch: the open source version of nginx, unfortunately, only resolves DNS records at start time and reload time. So if a DNS record you put in `dlan_proxy` (which is used in `upstream` blocks) changes after nginx starts, the only way you can have nginx update its upstream pool is to have it reload its config (e.g. `nginx -s reload`).

There's a trick that forces open source nginx to resolve records also over time (tl;dr: don't use upstreams, use variables inside of the `proxy_pass` directive), but it only works for HTTP proxying; as dlan uses TCP/UDP loadbalancing, we're out of luck.

##### Can i use a DNS record that points to multiple IPs in a `dlan_proxy` rule?
Yes, nginx supports that.

The `upstream` parameter is simply expanded to multiple `server` entries in the nginx config, for that service; and if a `server` entry is a record pointing to multiple IPs, then all of these are added to the upstreams pool.

##### How to listen on both IPv4 and IPv6 addresses, for a single `dlan_proxy` entry?
To do that, explicitly listen on both IPv4 and IPv6 addresses in your `listen` entries in `dlan_proxy`. E.g. to listen on all addresses for port 80, you should specify both `[::]:80` and `0.0.0.0:80`. This is because nginx has the `ipv6only` flag on by default, which means that when it binds to a port, it only uses one IP version.

The advantage of this is that you can expose multiple services using the same port, multiplexed by IP version (e.g. `0.0.0.0:80` loadbalances towards a certain service; `[::]:80` loadbalances towards a different service)


### Troubleshooting tips? Known issues? Gotchas?

- Some older versions of ansible (e.g. `2.8.1` is affected by this) may fail to cleanup the ansible tmpfile directory on the remote host, leading to the device filesystem filling up quickly. Upgrade your ansible installation to fix this (e.g. `2.9.9` is known to have fixed that).
- Under very specific circumstances, nginx may require a manual restart. This is required if you change the proxy entries, switching from a wildcard IP to a more specific IP for a certain port (e.g. you used to listen on `0.0.0.0:8080`, but now you listen on `192.168.1.5:8080`): nginx won't be able to actually perform this change until its full restart, as it needs to unbind from the port before being able to bind again to the more specific address.
- If you have traffic between multiple routers behind a single home network, but notice that the latency is unusually high, probably traffic is flowing through other locations and then goes back to the LAN where its destination is. In that case you should tune the OSPF link costs so that the local link is preferred.

Some OpenWrt packages that may be useful for debugging issues are: `ip-full`, `ip-bridge`
