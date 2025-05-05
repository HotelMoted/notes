

1.  **Proxmox SDN with EVPN (Ethernet VPN):**
    * **How it works:** Proxmox VE has a built-in Software-Defined Network (SDN) stack. One of its zone types is `EVPN`. This uses **FRRouting (FRR)**, a popular open-source routing suite (which includes BGP), on each Proxmox node under the hood. It combines VXLAN (for Layer 2 overlay) with BGP EVPN (for distributing MAC/IP information and handling Layer 3 routing).
    * **Mechanism:** When you configure an EVPN zone and attach VMs to its VNets (Virtual Networks), the SDN system, using FRR, automatically manages BGP announcements. When a VM starts or migrates to a node, the FRR instance on that node learns the VM's MAC/IP and can advertise reachability via BGP EVPN routes to your upstream BGP routers. When the VM stops or migrates away, the routes are withdrawn (or updated by the new host's FRR).
    * **Software:** Proxmox VE itself, `libpve-network-perl`, and `frr-pythontools` (install frr via `apt install frr-pythontools` and enable the service `systemctl enable frr.service`). Configuration is done via the Proxmox web UI under Datacenter -> SDN.
    * **Pros:** Integrated into Proxmox, managed via GUI/API, designed for cluster environments.
    * **Cons:** Can be complex to set up and understand (requires knowledge of BGP, EVPN, VXLAN), might have limitations depending on the exact desired behavior (e.g., fine-grained control over specific /32 announcements vs. subnet routing). Some forum posts suggest nuances in how specific VM IPs (/32 routes) are announced, especially regarding exit nodes.

Yes, software exists to do this.
* The **most integrated** way is using **Proxmox SDN with the EVPN zone type**, which leverages FRR internally.


**Overlay Networks (like VXLAN in Proxmox SDN):** This is crucial if the nodes are in different Layer 3 subnets. Technologies like VXLAN (used by Proxmox SDN's EVPN zones) create a Layer 2 overlay network *on top* of your existing Layer 3 infrastructure. This allows the VM to keep its original IP address and remain in the same logical Layer 2 network even when it physically moves between nodes that might reside in different subnets. The VXLAN tunnel handles bridging the Layer 2 traffic over the Layer 3 gaps.
5.  **Requirements:**
    * **BGP Peering:** Each Proxmox node needs to be able to establish a BGP session with your upstream router(s).
    * **Inter-Node Connectivity:** You still need reliable, reasonably low-latency network connectivity between all Proxmox nodes in the cluster for:
        * **Cluster Filesystem (pmxcfs):** Uses Corosync for communication (typically requires multicast or unicast over a dedicated, reliable link).
        * **Live Migration:** Needs significant bandwidth and low latency.
        * **Shared/Distributed Storage (if used):** Systems like Ceph are very sensitive to network latency and bandwidth between nodes.
    * **Overlay Network (if L3 separation exists):** If nodes are in different subnets, you need an overlay like VXLAN (provided by SDN EVPN or configured manually) for the VM networks themselves.

**In conclusion:** Yes, using Proxmox HA with dynamic BGP announcements (especially via SDN EVPN with VXLAN) is specifically designed to allow you to distribute your cluster nodes across different physical locations within a datacenter (different racks, potentially different network segments) while ensuring that VM IP addresses remain reachable and automatically follow the VM during migrations. This enhances fault tolerance and flexibility in node placement.





Okay, let's walk through exactly what would need to happen in the scenario where you **cannot use VXLAN/EVPN overlays** but **can use BGP**, and you want the VM (`192.168.10.5`) to work after migrating across the data center to a different access switch (implying a different L3 subnet).

We'll compare this setup to VXLAN/EVPN at the end.

**The BGP-Only Workaround Setup:**

1.  **Network/Hypervisor IPs:**
    * Node A is connected to Switch A, using access port for **VLAN 10**. Let's give Node A's NIC on this VLAN the IP `10.10.10.10/24`. The gateway for this subnet is `10.10.10.1`.
    * Node B is connected to Switch B (other side of DC), using access port for **VLAN 20**. Let's give Node B's NIC on this VLAN the IP `10.10.20.10/24`. The gateway for this subnet is `10.10.20.1`.
    * *(Note: These hypervisor interface IPs are crucial for the BGP next-hop strategy).*

2.  **VM IP Configuration:**
    * Remains static: IP `192.168.10.5`, Mask `255.255.255.0`, Gateway `192.168.10.1`. *(This config is still fundamentally mismatched with the networks Node A/B are on, which is the core problem we're working around).*

3.  **BGP Configuration (Handling Inbound Traffic):**
    * Your hypervisors (or a controller managing them) run BGP.
    * When VM is on Node A: Announce route `192.168.10.5/32` with **next-hop `10.10.10.10`**. Your core routers receive this, look up how to reach `10.10.10.10` (via VLAN 10), and send incoming packets for the VM onto VLAN 10 towards Node A. The packet arrives on the correct Node A interface.
    * When VM migrates to Node B: Announce route `192.168.10.5/32` with **next-hop `10.10.20.10`**. Routers now send incoming packets onto VLAN 20 towards Node B. The packet arrives on the correct Node B interface.
    * *Result:* Inbound traffic reaches the correct hypervisor on the correct interface.

4.  **Handling Outgoing Traffic (The Hard Part - ARP):**
    * The VM is now on Node B (connected to VLAN 20 / subnet `10.10.20.0/24`).
    * It needs to send traffic off-subnet (e.g., reply to incoming traffic, or initiate new connections). It looks up its gateway (`192.168.10.1`).
    * It tries to ARP for `192.168.10.1` by broadcasting on VLAN 20.
    * **ARP Fails:** Nothing on VLAN 20 has the IP `192.168.10.1`.
    * **The Workaround:** You **MUST configure a static ARP entry *inside the VM's guest operating system***. This entry maps the gateway IP to the hypervisor's MAC address on that segment.
        * Example command inside the VM (Linux): `sudo arp -s 192.168.10.1 <MAC_Address_of_Node_B_NIC_on_VLAN20>`
        * Now the VM *skips* the ARP broadcast and directly sends frames destined for its gateway IP to the hypervisor's MAC address.

5.  **Handling Outgoing Traffic (The Hard Part - Routing & Filtering):**
    * The VM sends a frame to the hypervisor (Node B) MAC address. Node B's network stack receives the IP packet inside (Src: `192.168.10.5`, Dst: e.g., `8.8.8.8`).
    * Node B needs **IP forwarding enabled** (`net.ipv4.ip_forward = 1`) to act as a router for this packet.
    * Node B uses its *own* default gateway (`10.10.20.1` on VLAN 20) to forward the packet.
    * The packet leaves Node B sourced from `192.168.10.5` onto the VLAN 20 segment.
    * **Potential Filtering:** The router (`10.10.20.1`) receives this packet. It might have **Unicast Reverse Path Forwarding (uRPF)** enabled. uRPF checks if the source IP (`192.168.10.5`) is reachable via the interface the packet arrived on. Since `192.168.10.5` isn't part of the `10.10.20.0/24` subnet expected on this interface, uRPF (in strict mode, often default) would **drop the packet**.
    * **The Workaround:** You **MUST disable or relax uRPF** on the gateway interfaces (`10.10.10.1` and `10.10.20.1`) to allow these packets with "wrong" source IPs through.

**Summary of BGP-Only Workaround Setup:**

* BGP used for dynamic `/32` announcements with next-hops matching the hypervisor's IP on the *local* VLAN/subnet.
* **Static ARP entries configured inside every migrating VM.**
* **IP forwarding enabled on hypervisors.**
* **uRPF disabled/relaxed on network gateways.**

---

**Comparison to VXLAN/EVPN Setup:**

Here's why the BGP-only workaround generally not good compared to VXLAN/EVPN:

1.  **VM Configuration Intrusiveness:**
    * **Workaround:** Requires manually injecting static ARP entries into the guest OS of *every* VM. This complicates deployment, templating, and management. What if the hypervisor MAC changes?
    * **VXLAN/EVPN:** Completely transparent to the VM. The VM uses standard DHCP or static IP configuration valid *within its overlay segment* and performs standard ARP for its *virtual* gateway (often an Anycast Gateway provided by EVPN across all nodes). **No special configuration inside the VM is needed.**

2.  **Network Security:**
    * **Workaround:** Requires **disabling uRPF**, a valuable security feature against IP spoofing, on your core network gateways. This is a significant security compromise.
    * **VXLAN/EVPN:** Does not require disabling uRPF on the physical network. The outer VXLAN packets use the hypervisors' legitimate IP addresses as source IPs, which pass uRPF checks. Security is maintained at the physical layer.

3.  **Complexity & Brittleness:**
    * **Workaround:** Creates a complex, brittle chain relying on BGP updates + correct static ARP + hypervisor routing + disabled router security + no further upstream filtering. A failure in any part is hard to diagnose across multiple independent systems.
    * **VXLAN/EVPN:** While having its own concepts to learn, it's a cohesive, standardized system. Troubleshooting follows established overlay/underlay methodologies. It encapsulates the complexity rather than spreading hacks across the stack.

4.  **Operational Overhead:**
    * **Workaround:** High overhead managing static ARP entries in potentially hundreds or thousands of VMs. Tracking which router interfaces have security weakened.
    * **VXLAN/EVPN:** Lower operational overhead once set up. The control plane automates endpoint learning and distribution. Adding/moving VMs is much simpler.

5.  **Standards vs. Hacks:**
    * **Workaround:** A non-standard collection of tricks to bypass fundamental L2/L3 behavior.
    * **VXLAN/EVPN:** Industry standard protocols (RFCs) designed precisely for this L2-over-L3 use case, ensuring better interoperability and leveraging common knowledge.

**In essence:** The BGP-only workaround forces complexity and configuration hacks *inside the VM* and requires you to *reduce network security*, whereas VXLAN/EVPN uses a dedicated overlay mechanism to handle the L2 abstraction *transparently* to the VM while maintaining underlying network security and standardization.


-------


Okay, here is a report structured for a Network Engineer, summarizing our discussion and outlining why Proxmox SDN with VXLAN BGP EVPN is the recommended approach for achieving seamless VM IP mobility across Layer 3 boundaries compared to alternative workarounds.

---

**Report: Achieving Seamless VM IP Mobility in Proxmox Across L3 Boundaries**

**Date:** May 5, 2025

**Prepared For:** Network Engineering Team

**Subject:** Analysis of solutions for maintaining static IP addressing for Virtual Machines (VMs) during migration across distinct Layer 3 network segments within a Proxmox VE cluster.

**1. Introduction / Problem Statement**

The primary objective is to enable seamless live migration of Virtual Machines (VMs) between Proxmox VE nodes located in different physical network segments (e.g., different racks connected via separate access switches, implying distinct VLANs and IP subnets) **without requiring the VM to be re-IPed**. The solution must maintain the VM's original IP configuration and network connectivity post-migration. Furthermore, the desired solution should avoid complex, per-VM manual configurations (like static ARP entries within the guest OS) and should integrate cleanly with Proxmox VE, ensuring persistence across software updates and upgrades without relying on custom patches.

**2. Challenges of Overlay-Less Mobility Across L3 Boundaries**

Attempting to migrate a VM with a static IP configuration (IP address, subnet mask, default gateway) to a hypervisor node physically connected to a different Layer 3 subnet inherently breaks standard IP networking from the VM's perspective.

* **L2/L3 Mismatch:** The VM's networking stack is configured for a specific L2 broadcast domain and L3 subnet. When placed onto a different physical segment, its configuration becomes invalid for that segment.
* **ARP Failure:** The most immediate impact is the failure of Address Resolution Protocol (ARP) for the VM's configured default gateway. The VM broadcasts ARP requests onto the new, incorrect L2 segment where its gateway IP does not exist, receives no reply, and is thus unable to send any traffic destined outside its original configured subnet.
* **BGP Limitation:** While BGP can be used to dynamically update routes for the VM's specific IP address (`/32` route), directing external traffic *to* the correct hypervisor node, it does not resolve the internal L2/L3 mismatch preventing the VM from functioning correctly on the new segment, particularly for initiating outbound connections.

**3. Analysis of the BGP-Only Workaround (Non-Overlay Method)**

A potential workaround exists using only BGP combined with specific configurations on the VM, hypervisor, and routers. This approach involves:

1.  **Dynamic BGP `/32` Routes:** Hypervisors announce the VM's `/32` IP using a next-hop IP address that is native to the local physical subnet/VLAN the hypervisor is currently connected to. This correctly handles *inbound* traffic delivery to the appropriate hypervisor interface.
2.  **Static ARP Entries (in Guest OS):** To bypass the inevitable ARP failure for the gateway, a static ARP entry mapping the VM's configured gateway IP to the MAC address of the local hypervisor's NIC must be manually configured **inside every migrating VM's operating system**.
3.  **Hypervisor IP Forwarding:** IP forwarding (`net.ipv4.ip_forward = 1`) must be enabled on the hypervisors, forcing them to act as routers for the VM's outbound traffic received via the static ARP mechanism.
4.  **Disabled Unicast Reverse Path Forwarding (uRPF):** The first-hop gateway routers must have uRPF security checks disabled or significantly relaxed on the interfaces facing the hypervisors. This is necessary because the hypervisor will be forwarding packets sourced from the VM's IP, which does not match the subnet the packet is egressing from, causing standard uRPF checks to drop the traffic.

**Significant Disadvantages of the BGP-Only Workaround:**

* **Guest OS Intrusion & Management Burden:** Requires manual static ARP configuration within each VM, complicating deployment, templates, and ongoing management (e.g., updates needed if hypervisor MACs change). This does not scale operationally.
* **Reduced Network Security:** Mandates disabling uRPF, a critical anti-IP-spoofing security feature, on core network infrastructure.
* **Complexity & Brittleness:** Creates a fragile dependency chain across BGP updates, VM configuration, hypervisor routing state, and router security settings. Troubleshooting is difficult due to the non-standard interactions. Relies on upstream networks not performing similar source-validation filtering.
* **Non-Standard Approach:** It's a collection of "hacks" rather than an engineered solution, making it difficult to support and maintain long-term.

**4. Recommended Solution: Proxmox SDN with VXLAN BGP EVPN**

The industry-standard and recommended approach leverages network virtualization via an overlay network, specifically VXLAN with BGP EVPN as the control plane, integrated within Proxmox SDN.

* **Mechanism:**
    * **VXLAN Overlay:** Creates virtual Layer 2 networks (identified by VNIs) that are tunneled over the existing Layer 3 physical network (the underlay). This decouples the logical network from the physical topology.
    * **BGP EVPN Control Plane:** Replaces inefficient flood-and-learn mechanisms. EVPN uses BGP extensions to dynamically distribute Layer 2 (MAC, MAC-IP) and Layer 3 (IP prefix) reachability information across VTEPs (VXLAN Tunnel Endpoints, which include the hypervisors).
    * **Proxmox Integration:** Proxmox SDN integrates FRRouting (providing BGP EVPN) and kernel VXLAN capabilities. Configuration is managed via the standard Proxmox API/GUI and persists across upgrades. Requires installing `frr-pythontools` and ensuring `libpve-network-perl` is present.

* **How it Solves the Problems:**
    * **L2/L3 Consistency:** The VM is always connected to its designated virtual network (VNI). Its IP configuration remains valid within this overlay, regardless of the physical location or subnet of the underlying hypervisor node.
    * **Standard ARP:** The VM performs standard ARP within the overlay network. EVPN facilitates features like ARP suppression or an Anycast Gateway (same gateway IP/MAC active on all nodes for a VNI), allowing ARP to resolve successfully without guest OS modification. **Static ARP entries are not needed.**
    * **Valid Outgoing Traffic:** The hypervisor encapsulates the VM's packets within VXLAN/UDP/IP headers. The outer source IP is the hypervisor's VTEP IP, which is a valid IP on the physical underlay network. This **passes standard uRPF checks** on physical routers.
    * **Seamless Mobility:** The combination allows VMs to migrate across L3 boundaries while retaining their IP addresses and network connectivity transparently.

* **Benefits:**
    * **VM Transparency:** Requires no modification to the guest OS networking stack.
    * **Enhanced Security:** Maintains underlay network security (uRPF intact). Provides robust L2/L3 segmentation via VNIs and optional VRF integration.
    * **Scalability:** Designed for large-scale deployments (up to 16M VNIs).
    * **Standardization & Robustness:** Leverages well-defined RFC standards, ensuring better interoperability and reliability.
    * **Manageability:** Centralizes network definition at the overlay level; leverages Proxmox integration. Reduces operational burden compared to per-VM hacks.
    * **Persistence:** Configuration is part of the Proxmox SDN framework and survives updates.

**5. Conclusion & Recommendation**

While BGP alone can be manipulated to direct traffic towards a migrated VM's new location, it fails to address the fundamental Layer 2 and Layer 3 requirements from the VM's perspective without resorting to complex, insecure, and operationally burdensome workarounds involving guest OS modifications (static ARP) and weakened network security (disabled uRPF).

**It is strongly recommended to implement Proxmox SDN utilizing VXLAN with BGP EVPN.** This approach provides a standardized, robust, scalable, and secure solution that directly addresses the requirement for seamless VM IP mobility across Layer 3 boundaries. It achieves this transparently to the guest VM, integrates cleanly with the Proxmox ecosystem ensuring persistence, and aligns with modern network virtualization best practices. While there is an initial learning curve associated with overlay networking concepts, the long-term operational stability, security benefits, and scalability far outweigh the complexities of maintaining the BGP-only workaround.

---
