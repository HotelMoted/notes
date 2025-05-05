Okay, thank you for that critical clarification. If the upstream switches cannot participate in EVPN and the Proxmox hosts connect via separate Layer 3 routed uplinks (each potentially leading to different upstream routers or routing domains), that definitely changes the *scope* of where the redundancy mechanisms operate.

Let's address the Network Engineer's concern with this topology in mind:

**Understanding the Setup:**

* **Proxmox Hosts:** Act as Software VTEPs, running VXLAN and BGP EVPN (via FRR, managed by Proxmox SDN).
* **EVPN Control Plane:** Runs **only between the Proxmox hosts** (or via host-based Route Reflectors). The upstream physical network is unaware of EVPN.
* **VXLAN Tunnels:** Established **between the Proxmox hosts** over the existing IP network provided by the separate routed uplinks and upstream routers.
* **Uplinks:** Each Proxmox host (or group of hosts) has one or more L3 connections to the upstream network (e.g., Host A connects to Router A, Host B connects to Router B). These upstream devices (Router A, Router B) do not speak EVPN.

**Addressing Redundancy:**

The engineer's point needs refinement, but touches upon a valid aspect related to **North-South traffic flow** in this specific scenario. Let's break down redundancy:

1.  **First-Hop Gateway Redundancy (Within the Overlay) - SOLVED by EVPN:**
    * As explained before, VXLAN BGP EVPN on the Proxmox hosts uses an **Anycast Gateway**. The *same* virtual gateway IP and MAC address exist on the relevant VNI interface on *all* participating Proxmox hosts.
    * When a VM (e.g., `192.168.10.5`) needs its gateway (`192.168.10.1`), it ARPs locally. The **local Proxmox host** it's running on responds immediately.
    * **Conclusion:** The VM *always* has a functional, reachable first-hop gateway provided by the host it resides on. This crucial part of redundancy **is handled correctly** by the host-based EVPN setup and **does not require multiple VM copies or IPs**.

2.  **Egress Path Redundancy (North-South Traffic Leaving the Overlay) - DEPENDS on Uplink/BGP Design:**
    * This is likely the core of the engineer's concern. When the VM sends traffic destined outside the VXLAN overlay (e.g., to the internet), the local Proxmox host receives it (acting as the Anycast Gateway). Now, that *host* must route the traffic out to the physical network.
    * Since the hosts have separate routed uplinks, Host A will typically route the traffic out via Router A, and Host B will route it out via Router B.
    * **Redundancy:** If Host A's uplink (or Router A) fails, can the traffic still get out?
        * The VM itself will likely be migrated by Proxmox HA to a healthy host (e.g., Host B).
        * Host B will then route the VM's traffic out via *its* uplink to Router B.
        * **Requirement:** For this to work seamlessly, the external network needs to learn that the path to the overlay subnets (or specific VM IPs if advertised) is now via Host B/Router B instead of Host A/Router A. This typically requires **BGP peering between the Proxmox hosts and the upstream routers** over these routed links, advertising the necessary overlay reachability information.
        * **Asymmetry:** Traffic might leave via Host B/Router B, but return traffic from the internet might still prefer a path coming in via Router A (if it's still partially up or based on external BGP metrics), leading to asymmetric flows. This can be managed but needs consideration, especially for stateful firewalls.
    * **Conclusion:** Redundancy for *exiting* the fabric relies on Proxmox HA moving the workload and correctly configured **standard BGP** (IPv4/IPv6 unicast, not EVPN) between the hosts and their respective upstream routers to handle path failover.

**Refuting the Engineer's Conclusion:**

* The statement "Your way won't help with upstream redundancy" is **too broad**. It *does* solve the critical **first-hop gateway redundancy** via the Anycast Gateway feature within the overlay.
* It *partially* addresses egress path redundancy by allowing workloads (VMs) to move to hosts with functional uplinks, provided the external BGP routing converges correctly.
* The claim that it requires "multiple copies of the vms with different IP configurations" to achieve redundancy is **incorrect**. The entire point of this setup (and overlays in general) is to maintain a *single, persistent IP* for the VM while providing gateway and potentially path redundancy underneath.

**How to Respond to the Engineer:**

"I understand the concern regarding redundancy, especially with separate routed uplinks and non-EVPN capable switches. However, the host-based VXLAN EVPN solution addresses redundancy at different layers:

1.  **First-Hop Gateway:** EVPN's **Anycast Gateway** feature ensures the VM *always* has a functional default gateway available locally on whichever host it runs on. This provides seamless L3 gateway redundancy *within the overlay* without needing multiple IPs for the VM.
2.  **Egress Path:** Redundancy for traffic *leaving* the overlay depends on our specific uplink topology and the standard BGP peering we configure between the Proxmox hosts and the upstream routers over those separate links. If a host or its primary uplink fails, Proxmox HA moves the VM, and BGP should converge to allow the VM's traffic to egress via the new host's available uplink. While this might involve managing potential path asymmetry, it provides failover without requiring duplicate VMs or IPs.

The core benefit here is solving the VM's L2/L3 requirements transparently via the overlay, separating that from the external path redundancy which is handled by standard BGP over the available uplinks."
