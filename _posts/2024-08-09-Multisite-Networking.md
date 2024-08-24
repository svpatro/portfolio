---
title: The Distributed Cyber Range
date: 2024-08-09
author: Shamik
---

### The Problem

One of the hardest problems when learning Cybersecurity is finding the right place to train. While some might resort to virtual machines on their computers, others might look to solutions in public cloud. For our Collegiate Cybersecurity Defense Competition (CCDC) Team, we built our own solution.

### The Approach

Initially, the thought was to self-host a cluster of servers at a teammate's apartment. However, we quickly realized that network bandwidth would become an issue, let alone the sheer amount of power draw that would result from having the servers on for an extended period of time. Turns out most residential circuit breaks are designed to trip at 15 amperes, meaning that consistent power draw from a server working near 100% utilization would easily trip the circuit and risk frying the power supply units. 

What was truly needed for our idea to work was geographically distributing a set of servers while having the network services accessible from anywhere. Typically, this requires a site-to-site virtual private network (S2S VPN) with layer-3 routing devices.

One might ask, "Why not install a VPN client into every single computer into the VPN network?"

The reason is three-fold: 
1. Configuration time would be proportional to how many computers are needed for each training session.
2. Having a dedicated Layer-3 virtual routing device allows us to create our own virtual LANs, allowing for a more secure and segmented network overall.
3. Many layer-3 routing devices are internet gateways and have built-in VPN's, making our lives much easier.

### Multisite Networking

For our setup, we used [Tailscale](https://tailscale.com/) for our VPN. Tailscale is a centralized VPN service, meaning that all traffic will flow through their servers before hitting our own. However, the traffic is end-to-end encrypted and you don't need to be connected to the VPN to see if your devices are connected to the Tailscale network. Tailscale also has a neat feature where you can configure any device in your network to act as as internet gateway (known as a [subnet router](https://tailscale.com/kb/1019/subnets)).

For our servers, we utilized several pre-owned Dell servers from [SaveMyServer](https://savemyserver.com/). Once recieved, we flashed [Proxmox Virtualization Environment](https://www.proxmox.com/en/proxmox-virtual-environment/overview) onto all of servers for a readily accessible user interface. 

For our layer-3 router, we decided to use a virtualized version of Netgate's [PFSense](https://www.pfsense.org/), which conveniently has the Tailscale package readily available. Setting up each virtualized router required us to point to our local router's as the upstream router (WAN) and create a virtualized switch (Virtual Machine Bridge) to connect any new machine into our virtualized LAN. 

In order to have each remote site be able to communicate to each other, all virtualized PFSense routers would need to be in the same Tailscale instance.

![Tailscale Network](/assets/img/Range.png)


### Challenges Faced

While having the Tailscale package developed for PFSense was a lifesaver, the package itself does not support disabling SNAT (source network address translation). As a result, packets are rewritten with Tailscale's translated IP when arriving to the destination, causing havoc on the packet's return trip. Through some fanagling with PFSense's virtual IP / outbound NAT feature, this can be resolved to allow true site-to-site traffic flow.

Hosting in multiple locations also means turning on and off multiple servers remotely to save on power consumption. This requires having a dedicated low-powered device on the network that is able to access the server's management page whenever needed. As a result, a dedicated tailscale account (tailnet) is needed per remote site since only one tailnet is allowed per account.