Yes, that's exactly right!

If you lack the necessary licenses to run BGP EVPN directly **on your switches**, you can absolutely shift that functionality to your **Proxmox hosts**, making them act as **Software VTEPs (VXLAN Tunnel Endpoints)**.

Here's how it works:

1.  **Proxmox Uses FRRouting (FRR):** Proxmox VE doesn't have EVPN built into its core virtualization code. Instead, it leverages a powerful, standard open-source routing suite called **FRRouting (FRR)**. FRR is a fork of Quagga and implements many routing protocols, including BGP and its EVPN address family extension.
2.  **FRR Runs on Proxmox Hosts:** You install FRR (typically the `frr` or `frr-pythontools` package) on each Proxmox node in your cluster.
3.  **EVPN Control Plane on Hosts:** The FRR instance running on each host handles the BGP EVPN control plane tasks:
    * Peering with other Proxmox hosts (or dedicated route reflectors) using BGP.
    * Advertising the MAC addresses (and optionally IP addresses) of locally connected VMs using EVPN Type-2 routes.
    * Learning about remote MAC/IP addresses and their corresponding VTEP IPs from other hosts via BGP EVPN updates.
    * Potentially handling L3 routing information using EVPN Type-5 routes if you configure IRB (Integrated Routing and Bridging).
4.  **VXLAN Data Plane on Hosts:** The Linux kernel on the Proxmox host handles the actual VXLAN data plane encapsulation and decapsulation. FRR interacts with the kernel to manage the necessary forwarding state based on the EVPN control plane information.
5.  **Proxmox SDN Role (Optional but Recommended):** The Proxmox SDN stack provides a management layer **on top of FRR and the kernel's VXLAN features**. When you configure an EVPN zone, VNets, etc., through the Proxmox GUI or API:
    * Proxmox SDN automatically generates the correct `frr.conf` configuration snippets for BGP EVPN.
    * It configures the necessary `vxlan` interfaces and bridges using Linux networking tools.
    * It ensures this configuration is distributed across the cluster and **persists across Proxmox updates/upgrades**.

**In Summary:**

Yes, **Proxmox itself doesn't *contain* EVPN, but it integrates and manages FRRouting (FRR), which *does* implement BGP EVPN**. By installing FRR and configuring it (ideally via the Proxmox SDN interface), your Proxmox hosts become fully functional Software VTEPs capable of running both the VXLAN data plane and the BGP EVPN control plane. Your physical switches then only need to provide the basic Layer 3 IP connectivity (the underlay) between the hosts.
