# dlan

* [Overview](#dlan)
* [Installation and usage](#installation-and-usage)
* [Architecture and design](./docs/architecture.md)
* [F.A.Q.](./docs/faq.md)
* [Future work](#future-work)

dlan (aka Distributed LAN) is an ansible role to setup a geographically distributed private network, for selfhosting your services. A set of routers (OpenWrt nodes) will act as a virtual perimeter, separating your _internal network_ (spanning multiple locations) from the rest of the web, and securely exposing services running behind it.

In a nutshell, dlan lets you build a WAN that mimics the semantics of a home LAN.

[Overview of a dlan setup](./docs/overview.svg)

Features:
- Transparent, private L3 connectivity among internal hosts. Routers will take care of creating an encrypted overlay network (wireguard, VXLAN, OSPF), so that you can just plug a host into a LAN port of any of the routers, to make it securely reachable from any other host and router in the dlan.
- Expose internal services through nginx TCP/UDP loadbalancing, from any dlan router to any internal host, no matter the router's or host's geographical location.
- Partial mesh connectivity of the routers, and OSPF with tunable link costs. You can implement a network resilient to partial failures, and able to prefer local links for minimizing latency.
- One-way peering: a dlan router can live in a private network that you don't control (e.g. you're not allowed to create port forwardings on your home router, because it's a shared house or there may be other technicians that may mess your configuration), and still join the network by peering to another publicly reachable router.
- Do not trust anything beyond the WAN port: dlan routers act as your internal network perimeter, and thus are designed to stand between your servers and untrusted networks. E.g. even if you have multiple dlan routers in the same home LAN, they will assume that their upstream home network is untrusted, and thus encrypt traffic destined to each other.
- Netowrk features: DHCPv4 and DHCPv6 with optional static leases, DNS server, IPv6 support (ULA only)
- Runs on commodity hardware.
- OpenWrt based (may change in the future)


### Installation and usage

Important note: changes to UCI config are performed directly on-disk, and daemons are restarted right after. If there is an issue in the config, you may risk making the device unreachable. It's recommended to double-check the produced configurations before applying, and getting familiar with failsafe mode in case things go wrong. Previous experience in Linux administration and OpenWrt is also recommended.

Prerequisites:
- One or more OpenWrt routers (>=19.0.2)
- Passwordless root access to the router
- The router can access the internet

Please check the [defaults file](./defaults/main.yml) for a full list of documented role variables.
Also the [dlan-example](https://github.com/mvitale1989/dlan-example) repo provides a ready-to-use VM-based testbed for trying this project out.



### Future work

Functional improvements:
- Multiple public IPs for a single WAN port (see TODO in `defaults/main.yml`)
- Create automatioun around nginx reloads, on DNS record changes
- Add reverse DNS entries
- Support internal CNAME/SRV/TXT DNS Records
- Support PXE boot option control
- Make it possible to inject non-dlan VPN client keys (e.g. you may want to use the VPN from a laptop)
- Add rules to have traffic from a certain VPN client go out of a certain router

Architectural changes:
- Support debian linux machines, in addition to (or instead of) OpenWrt
- Support distributed L2 segments and multihoming, e.g. EVPN (e.g. frrouting) or an OpenFlow SDN (e.g. openvswitch)

Nice-to-haves and cleanups:
- Use only one VXLAN interface per-router, instead of one for each pair of routers, using static FIB entries, if possible
- Autogenerate `dlan_id`, instead of having the user set them explicitly (e.g. generate from the inventory name)
- Generate IPs for the current host (e.g. loopack/wg ipv4/v6) once, in a single place (e.g. `default/internal.yml`), instead of computing them at every place
- Use `dlan_maxhosts` to generate all subnet masks
- Have a user other than `root` be the one for connecting, for extra security against bruteforcing
- Write about the project's security model (e.g. is my internal network safe if one of the dlan routers runs on EC2?)
