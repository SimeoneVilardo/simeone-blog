---
author: ["Simeone Vilardo"]
title: "Advanced WireGuard on Docker: Split Tunneling, No-NAT, and Hardening on Modern Linux"
date: "2025-01-20"
description: "Configuring secure WireGuard for a homelab"
summary: "Configuring WireGuard via Docker (using popular images like wg-easy) is often marketed as a 5-minute operation. And it is, if your only goal is a working tunnel without worrying about what happens 'under the hood.' However, when you have specific engineering requirements—such as integration with a local DNS (e.g., Pi-hole), the need for real logs (No-NAT), and granular security (Firewalling)—the out-of-the-box configuration reveals its limitations, especially on modern Linux distributions (Debian 12/13, Ubuntu 22.04+) that utilize nftables and a complex Docker subsystem."
tags: ["networking", "homelab"]
series: ["Network", "Homelab"]
---

# Advanced WireGuard on Docker: Split Tunneling, No-NAT, and Hardening on Modern Linux

Configuring WireGuard via Docker (using popular images like `wg-easy`) is often marketed as a 5-minute operation. And it is, if your only goal is a working tunnel without worrying about what happens "under the hood."

However, when you have specific engineering requirements—such as integration with a local DNS (e.g., Pi-hole), the need for real logs (No-NAT), and granular security (Firewalling)—the out-of-the-box configuration reveals its limitations, especially on modern Linux distributions (Debian 12/13, Ubuntu 22.04+) that utilize `nftables` and a complex Docker subsystem.

In this article, we will analyze how to build a robust **Split Tunnel VPN** architecture, resolving conflicts between `iptables-legacy` and `nftables`, and correctly managing packet flow through Docker chains.

## 1. The Architectural Objective

We aim to achieve an environment where:

1. **Real Split Tunnel:** Clients send **only** traffic destined for internal services and DNS through the tunnel. Everything else (Internet) travels over the client's connection (4G/Wi-Fi).
2. **Observability (No-NAT):** The DNS server (Pi-hole) must see the VPN client's real IP (e.g., `10.13.13.2`), not the Docker gateway IP.
3. **"Least Privilege" Security:** Even if a client manipulates its configuration to tunnel all traffic, the server must discard (DROP) anything not explicitly authorized.

## 2. Client-Side Configuration: `WG_ALLOWED_IPS`

The first level of traffic management occurs on the **Client**. WireGuard does not have a dynamic route "push" mechanism like OpenVPN; routes are defined statically in the peer configuration.

In the `wg-easy` `docker-compose.yml`, we use the `WG_ALLOWED_IPS` variable to generate client configs:

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy
    network_mode: host
    environment:
      # Internal VPN Subnet
      - WG_DEFAULT_ADDRESS=10.13.13.x
      # DNS IP (Pi-hole)
      - WG_DEFAULT_DNS=192.168.0.2
      
      # FUNDAMENTAL: Split Tunneling
      # Instructs the client to create tunnel routes ONLY for these IPs/Ranges.
      # If we put 0.0.0.0/0 here, the client would lose internet connectivity
      # (because our server-side firewall will block exit to WAN).
      - WG_ALLOWED_IPS=192.168.0.2/32,192.168.0.4/32,10.13.13.0/24

```

**Technical Note:** This configuration resides on the user's device. A malicious (or curious) user can manually modify it on their smartphone by setting `AllowedIPs = 0.0.0.0/0`. For this reason, we cannot rely solely on this configuration: a strict server-side firewall is required (see Section 5).

## 3. Visibility and Routing: Why Kill Masquerading

The default `wg-easy` configuration enables **Masquerading (NAT)**. This means the VPN server replaces the packet's source IP (`10.13.13.2`) with its own LAN IP (`192.168.0.4`) before forwarding it.

* **Pro:** No configuration required on the local network.
* **Con:** Pi-hole logs become useless (everything appears to come from the VPN server).

**The No-NAT Solution:**
We disable masquerading (we will see how in Section 4). However, without NAT, when a packet arrives at the Pi-hole with sender `10.13.13.2`, the Pi-hole will try to reply by sending the packet to its Default Gateway (the Internet Router), which will discard the packet because it knows nothing about the `10.13.13.0/24` network.

We must add a **Static Route** on the LAN Gateway (the home router) or directly on the target hosts (Pi-hole):

> *Destination Network:* `10.13.13.0/24` -> *Next Hop (Gateway):* `192.168.0.4` (VPN Server IP).

Additionally, on Pi-hole (Settings -> DNS), you must enable **"Permit all origins"**; otherwise, queries coming from a different subnet (`10.x.x.x`) will be ignored due to the default security policy.

## 4. The Docker Paradox and the "Split Brain" Firewall

This is the core of the technical issue on modern Linux systems.

### The Problem: Legacy vs. NFT

The `wg-easy` image attempts, at startup, to inject permissive rules using `iptables` binaries inside the container. These are often `iptables-legacy`. The host (Debian 13) uses `iptables-nft` (a wrapper for nftables). This creates two disjoint sets of rules that can conflict or be ignored.

### The Problem: Default Rules

By default, `wg-easy` executes commands similar to these:

```bash
iptables -A FORWARD -i wg0 -j ACCEPT
iptables -A FORWARD -o wg0 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.13.13.0/24 -j MASQUERADE

```

If we want to manage security and remove NAT, we must inhibit this behavior. We do this by "hacking" the `WG_POST_UP` variable in the compose file:

```yaml
    environment:
      # A dummy command that prevents the execution of default scripts
      - WG_POST_UP=echo "Skipping default rules"
      - WG_POST_DOWN=echo "Nothing to do"

```

### The Block: Why does connectivity die?

As soon as we remove the automatic rules mentioned above (especially the first two `ACCEPT` rules), the VPN stops working towards Docker containers. Why?

Docker, to isolate containers, sets the **FORWARD chain Policy to DROP**.

When a packet arrives from `wg0` (a network not managed by Docker) directed to a container:

1. **Without explicit rules:** Docker doesn't recognize the interface, iterates through its chains, finds no match, and applies the final `DROP`.
2. **With Masquerading (if it were active):** The packet would appear to come from the Host IP or Gateway IP. Docker tends to trust "local" traffic, so it often lets NATed packets pass. This is why NAT often magically "solves" problems: it hides the external origin of the traffic.

Since we **do not want to use NAT**, the packet presents itself with the alien IP `10.13.13.x` and is brutally discarded by Docker's policy.

### The Solution: `DOCKER-USER` Chain

Docker provides a specific chain, called `DOCKER-USER`, which is evaluated **before** internal blocking rules. We must insert our exceptions here.

It is crucial to use `-I` (Insert) and not `-A` (Append) to ensure the rule is at the top of the list.

```bash
# Allow VPN packets to enter Docker routing
sudo iptables -I DOCKER-USER -i wg0 -j ACCEPT

# Allow responses to return
sudo iptables -I DOCKER-USER -o wg0 -j ACCEPT

```

*Note: These commands must be made persistent (e.g., via systemd or rc.local) as Docker resets chains upon restart.*

## 5. NFTables: The Final Whitelist (Server-Side)

Now that we've opened the Docker gates via `DOCKER-USER`, traffic flows. But it flows *too much*. Without further filtering, a VPN client could try to reach anything.

We must configure `nftables` on the host to apply a strict **Whitelist logic**.

File `/etc/nftables.conf`:

```nft
#!/usr/sbin/nft -f

flush ruleset

table inet my_shield {
    # INPUT Chain: Protects the Host itself
    chain input {
        type filter hook input priority 0; policy accept;
        
        # Accept VPN traffic only towards the server IP (e.g., 192.168.0.4)
        iifname "wg0" ip daddr 192.168.0.4 accept
        
        # Block attempts to access other host ports/services
        iifname "wg0" drop
    }

    # FORWARD Chain: Handles passing traffic (towards Docker, LAN, Internet)
    chain forward {
        type filter hook forward priority 0; policy accept;

        # Accept established connections (necessary for packet return)
        ct state established,related accept

        # --- CRITICAL RULE FOR DOCKER ---
        # When you access a container (e.g., port 8080), Docker performs DNAT to an internal IP (172.x...).
        # This rule explicitly accepts "NATed" traffic (destined for containers).
        iifname "wg0" ct status dnat accept

        # Access to Pi-hole (Physical IP)
        iifname "wg0" ip daddr 192.168.0.2 accept
        
        # Direct access to server (Security redundancy)
        iifname "wg0" ip daddr 192.168.0.4 accept

        # --- FINAL DROP ---
        # If a client tries to go to the Internet or access other LAN IPs, it is blocked here.
        iifname "wg0" drop
    }
}

```

## Conclusion

We have replaced the "magic" of automatic configurations with a deep understanding of packet flow.
The result is a system where:

1. **Docker** accepts raw VPN packets (thanks to `DOCKER-USER`).
2. **Routing** handles packet return (thanks to static routes, without NAT).
3. **NFTables** acts as the final guardian, blocking any connection not strictly necessary.

This setup is not just "working"; it is **observable** and **secure by design**.
