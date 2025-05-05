Okay, that's a crucial constraint. Lacking the licensing for BGP EVPN **on the switches** means the "Network Centric" Hardware VTEP model using EVPN as the control plane is off the table. Your Arista switches cannot provide that intelligence layer in this case.

This brings us back to evaluating the other possibilities. Let's reassess based on this new information:

**Option 1: Hardware VTEP (Switches) with Manual VXLAN Control Plane**

* **Possibility:** Can your Arista switches perform basic VXLAN L2 Gateway functions (map VLAN to VNI, encapsulate/decapsulate) and use either **Multicast** or **Static Head-End Replication (HER)** for the control plane *without* requiring the EVPN license?
    * *Requires Check:* You would need to verify the specific licensing requirements for your EOS version (4.18.11M) on the DCS-7050QX. Basic VXLAN bridging and potentially static replication *might* be included in base licenses, but multicast (PIM) often requires specific licensing. Check Arista documentation or feature navigator.
* **If Yes (Static HER Possible on Switch):**
    * *Setup:* Configure VXLAN interfaces, VLAN-VNI mappings, and *manually define the IP addresses of all other switch VTEPs* on each switch for the relevant VNIs.
    * *Proxmox Config:* None needed for VXLAN/EVPN. Hosts connect to standard VLANs.
    * *Downside:* High configuration overhead managing the static peer lists on switches, especially as your fabric grows. Scalability is poor.
* **If Yes (Multicast Possible on Switch):**
    * *Setup:* Configure VXLAN interfaces, VLAN-VNI mappings, and enable/configure PIM multicast routing in your underlay network. Map VNIs to multicast groups.
    * *Proxmox Config:* None needed for VXLAN/EVPN.
    * *Downside:* Requires implementing and managing multicast routing in your network, which has its own complexity. Flood-and-learn MAC discovery is less efficient than EVPN.
* **If No (Basic VXLAN/HER/Multicast Needs License):** This option is out.

**Option 2: Software VTEP (Proxmox Hosts) with BGP EVPN**

* **Mechanism:** This shifts the VTEP function and the EVPN control plane entirely to the Proxmox hosts. The hosts run FRR (for BGP EVPN) and use the kernel for VXLAN encapsulation. Your Arista switches only need to provide the Layer 3 underlay connectivity.
* **Proxmox Config:** **Required.** You need to set up VXLAN interfaces, bridges, and FRR with BGP EVPN configuration on each Proxmox node.
    * **Using Proxmox SDN:** This is the **recommended way** to implement this option. Proxmox SDN provides an integrated framework to configure and manage FRR and the associated VXLAN networking across the cluster via the GUI/API. This configuration **will persist** across Proxmox updates.
    * **Manual Configuration:** You *could* manually install FRR, configure `frr.conf`, set up `ip link` VXLAN interfaces, manage bridges via `/etc/network/interfaces` etc., outside of the Proxmox SDN stack. This gives ultimate control but loses the integration benefits and you are responsible for ensuring persistence and consistency.
* **Licensing:** No special Arista licenses needed beyond basic L3 routing.
* **Downsides:** Consumes some CPU/RAM on hypervisors for VTEP & BGP processes. Configuration occurs on the hosts.

**Option 3: Software VTEP (Proxmox Hosts) with Manual VXLAN Control Plane**

* **Mechanism:** Proxmox hosts are VTEPs, configured manually (not Proxmox SDN). They use either Multicast or Static HER *between themselves* for the control plane.
* **Proxmox Config:** **Required.** Manual setup of VXLAN interfaces/bridges + either configuring multicast listener/sender logic or manually maintaining static HER peer lists *on every Proxmox host*.
* **Licensing:** No special Arista licenses needed.
* **Downsides:** Same control plane issues as Option 1 (Multicast complexity or HER burden), but now implemented and managed directly on the hosts.

**Option 4: BGP-Only Workaround (Static ARP, Disabled uRPF)**

* **Mechanism:** As discussed before. BGP on hosts for `/32`, static ARP inside VMs, IP forwarding on hosts, disabled uRPF on routers.
* **Licensing:** No special Arista licenses needed.
* **Config:** Requires changes/config on BGP (hosts), **VMs (Static ARP)**, Hypervisors (IP Forwarding), and **Routers (uRPF disabled)**.
* **Downsides:** Remains the least recommended due to **security risks (disabled uRPF)**, **intrusive VM configuration (static ARP)**, complexity, and brittleness.

**Revised Recommendation (No Switch EVPN License):**

1.  **Investigate Switch Capabilities:** First, **verify Arista licensing** for basic VXLAN L2 Gateway functionality with Static Head-End Replication on your EOS version. If this is available without the EVPN license and the management overhead of static peer lists on the switches is acceptable, this (Option 1B) might be the simplest path *for the hosts*. If multicast is licensed and preferred over HER, Option 1A is also viable.
2.  **Recommended Host-Based Solution:** If Hardware VTEP without EVPN isn't feasible or desirable, the next best approach is **Software VTEP on Proxmox Hosts using BGP EVPN, managed via the Proxmox SDN stack** (Option 2, Proxmox SDN variant).
    * This uses a proper overlay and a modern control plane.
    * It avoids static ARP in VMs and disabling uRPF.
    * The Proxmox SDN integration provides easier management and ensures the configuration persists through Proxmox updates/upgrades, addressing your concern about custom patches being overwritten.
3.  **Fallback Host-Based:** Manual host-based VXLAN with Multicast/HER (Option 3) is possible but generally less desirable than using EVPN via Proxmox SDN due to control plane limitations or management overhead.
4.  **Last Resort:** The BGP-only workaround (Option 4) should be avoided if possible due to its significant drawbacks.

Therefore, since you lack switch EVPN licenses, focusing on **Option 2 using the integrated Proxmox SDN stack** likely represents the best balance of functionality, maintainability, security, and scalability.
