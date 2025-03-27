---
title: The Distributed Cyber Range
date: 2024-08-09
author: Shamik
---

### The Problem

One of the hardest problems when picking up Cybersecurity is just finding the right place to train. While some might resort to virtual machines on their physical computers, others might look to solutions in public cloud. For our Collegiate Cybersecurity Defense Competition (CCDC) Team, we built our own solution.

### The Approach

Initially, the thought was to self-host a cluster of servers at a teammate's apartment. However, we quickly realized that network bandwidth would become an issue, let alone the sheer amount of power draw that would result from having the servers on for an extended period of time. Turns out most residential circuit breaks are designed to trip at 15 amperes, meaning that consistent power draw from a server working near 100% utilization would easily trip the circuit and risk frying the power supply units. 

What was truly needed for our idea to work was geographically distributing a set of servers while having the network services accessible from anywhere. Typically, this requires a site-to-site virtual private network with layer-3 routing devices.

One might ask, "Why not install a VPN client onto every single computer in the VPN network?"

Quite a few reasons, actually: 
1. Configuration time would be proportional to how many computers are needed for each training session.
2. Having a dedicated Layer-3 virtual routing device allows us to create our own virtual LANs, allowing for a more secure and segmented network overall.
3. Many layer-3 routing devices are internet gateways and have built-in VPN's, allowing any traffic to flow specifically through the VPN tunnel.
4. Virtual machines are stationary on Type I hypervisors, so gateways will always be able to communicate with the machines and vice versa.
5. More VPN clients adds complexity and increases pricing dramatically (remember, we're still in college!)

### Multisite Networking

For our setup, we used [Tailscale](https://tailscale.com/) for our VPN. Tailscale is a centralized VPN service, meaning that all traffic will flow through their servers before hitting our own. However, the traffic is end-to-end encrypted and you don't need to be connected to the VPN to see if your devices are connected to the Tailscale network. Tailscale also has a neat feature where you can configure any device in your network to act as the gateway for the network (known as a [subnet router](https://tailscale.com/kb/1019/subnets)).

```bash
# Short script to transform an LXC into a VPN endpoint with subnet routing enabled.
# Detailed instructions available here: https://github.com/svpatro/Subnet-Router/
apt-get remove openssh-server -y
ufw deny 22
apt update && apt upgrade -y
apt install curl -y
curl -fsSL https://tailscale.com/install.sh | sh
touch start_vpn.sh
echo "tailscale up --advertise-routes=<routes> --accept-routes" > start_vpn.sh
chmod +x start_vpn.sh
```

For our physical servers, we utilized several pre-owned Dell servers from [SaveMyServer](https://savemyserver.com/). Once recieved, we flashed [Proxmox Virtualization Environment](https://www.proxmox.com/en/proxmox-virtual-environment/overview) onto all of servers for an open-source and readily accessible network service interface. 

For our layer-3 router, we decided to use the [PFSense](https://www.pfsense.org/) virtual router, which conveniently has the Tailscale VPN package readily available. Setting up each virtualized router requires us to point to our local router's as the upstream router (WAN) and create a virtualized switch (Virtual Machine Bridge - VMBRx) to connect any new machine into a virtualized LAN. 

In order to have each remote site be able to communicate with one other, all virtualized PFSense routers would need to be in the same Tailscale instance. This effectively builds the multisite VPN setup since traffic is able to flow from site-to-site and internal sites are served as long as a client is within the virtualized LANs.

Notice how any VM is able to communicate with any other VM by virtue of the layer 3 devices in the Tailscale network:

![Tailscale Network](/assets/img/Cyber_Range.jpg)
_Note: Some IP's are removed for the sake of privacy. Green devices in the VLANs are VMs that advertise specific services on the network._


### Challenges Faced

- While having the Tailscale package developed for PFSense was a lifesaver, the package itself does not support disabling SNAT (source network address translation). As a result, packets are rewritten with Tailscale's translated IP when arriving to the destination, wreaking havoc on the packet's return trip. Through some fanagling with PFSense's virtual IP / outbound NAT feature, this can be resolved to allow true site-to-site traffic flow.

- Hosting in multiple locations also means turning on and off multiple servers remotely to save on power consumption. This requires having a dedicated low-powered device on the network that is able to access the server's management page whenever needed. As a result, a dedicated tailscale account (tailnet) is needed per remote site since only one tailnet is allowed per account.