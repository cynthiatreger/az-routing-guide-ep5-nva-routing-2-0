### [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
##
# Episode #5: NVA Routing 2.0 with Azure Route Server, VxLAN (or IPSec) & BGP

*Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.*
##
[5.1. ccc]()

[5.2. dds]()

&emsp;[4.2.1. vvv]()
##
# Previously...

In the 2 last episodes we enabled end-to-end connectivity between Azure and On-Prem branches connected to a non-Azure-native Concentrator ([Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals)) and then added FW inspection via a non-Azure-native Firewall ([Episode #4](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas)).

We demonstrated that OS-level routing was not enough and that the NIC *Effective routes* of all the VMs in the local and peered VNETs had to be aligned to the NVA routing tables using UDRs. These UDRs had to be applied on every subnet, using separate Azure *Route tables* per environment (Spoke, FW NVA, Concentrator NVA), without any central point of configuration.

For larger deployments with tens or even a few hundreds of Spoke ranges and a variety of On-Prem prefixes, not always strictly aggregable under RFC1918, this can quickly become complex to both implement and operate.
##
# 5.1. Benefits of Azure Route Server (ARS)

The [Azure Route Server](https://learn.microsoft.com/en-us/azure/route-server/overview) (ARS) is an Azure resource deployed in a dedicated subnet (named *RouteServerSubnet*) that acts like a Route Reflector and whos job is basically to align the control plane (NVA routing table) and the data-plane (*Effective routes*).

When deployed, the ARS can run BGP with an NVA and as a result will dynamically propagate and program the routing information received from this NVA in the *Effective routes* of all the VMs in the local and peered VNETs.
The ARS also works the other way round and advertises these VNET ranges to its BGP-peered NVA.

When the ARS is used in a VNET with an Azure Virtual Network Gateway (VPN or ER), iBGP is automatically run between them to provide, if required, branch-to-branch transit (OFF by default).

The ARS takes over the control from the Azure Virtual Network Gateways if any, and becomes the new controler of the Azure scope. Consequently, the *GW Transit* and *GW route propagation* settings apply to ARS, as we will see in the next section.

⚠️ Please note the ARS is NOT in the data-path. For non-Azure-native scenarios, the ARS is paired with an NVA, and then attracts and forwards traffic through this NVA.

In this episode we are going to address one of the many solutions ARS provides. If you would like to find out more about the routing scenarios leveraging ARS, please check out Mays' [ARS MicroHack](https://github.com/malgebary/Azure-Route-Server-MicroHack). There is also Adam's great [video on ARS placement](https://youtu.be/eKRuJPjCR7o), for which I probably owe 50% of the current views. 

# 5.2. Single NVA and ARS (Episode #3-like topology)

To illustrate the impact and power of ARS, we will start with an Episode #3-like environment, updated with an ARS: 
- 1 Hub VNET peered with 2 Spokes (Spoke1 and Spoke2)
    - *GW transit* enabled for Spoke1
    - *GW route propagation* disabled on Spoke1/subnet2
    - *GW transit* disabled for Spoke2
- 1 Concentrator NVA for On-Prem branch connectivity
- 1 ARS peered with the Concentrator NVA

*All the *Route tables* and UDRs configured in the previous episodes have been removed, as well as the 10/8 static route configured on the Concentrator and advertised On-Prem for Azure reachability.*

<img width="1120" alt="image" src="https://user-images.githubusercontent.com/110976272/216853300-9a5e1a53-b066-4e37-99bb-6b6a5795a72e.png">

💡The whole Episode #3 lab has been completed in one step: On-Prem reachability has been propagated to the Hub VNET (hosting the ARS) and its the peered Spoke VNETs (like a Virtual Network Gateway would do), without any UDRs. 

With just ARS and BGP running with the Concentrator we obtained automatic route propagation and programmation both at the NIC level and OS level for traffic steering through the Cocnentrator NVA: 
- The *Effectives routes* of the VMs in the local VNET and peered VNETs contain the On-Prem branch prefixes, pointing to the Concentrator NVA NIC.
- The Concentrator NVA routing table contains the appropriate VNETs ranges, that are further propagated to On-Prem.

➡️ *GW transit* and *GW route propagation* are honored for NVA advertised prefixes as shown by the routes advertised from the ARS to the Concentrator NVA (Spoke2 range missing) and the *Effective routes* of Spoke1VM2 and Spoke2VM and confirmed by the failed pings.

For the rest of this episode, we will focus on Spoke1/subnet1 only (both *GW Transit* and *GW route propagation* ON).

# 5.3. Chained NVAs and ARS (Episode #4-like topology)

Without ARS, in chained NVAs scenarios (like in [Episode #4](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas)), we have seen that the routing information shared between the 2 NVAs OS is NOT available to the NVAs *Effective routes*: the potential benefit of running dynamic routing (BGP) between the NVAs is taken away by the heaviness of UDRs required globally (Spoke subnets, FW subnet, Concentrator subnet), for every single Spoke range and On-Prem prefix.

*The FW bypass use case is left out of this lab.*

## 5.3.1. Chained NVAs and direct ARS plugin (and UDRs...)

Let's look at the impact of adding an ARS in our previous chained NVA test environment. 

### 5.3.1.1. Initial connectivity diagram

As the goal is now to steer traffic through the FW NVA for inspection, we want the ARS to program the VMs so their *Effective routes* for On-Prem prefixes point to the FW NVA. Having Next-Hop = FW NVA (10.0.0.5) automatically set for On-Prem connectivity means the ARS is peered with the FW NVA and is receiving these On-Prem prefixes via BGP.

<img width="1148" alt="image" src="https://user-images.githubusercontent.com/110976272/217000340-6da77cdb-fbbb-46e2-8f14-d6ed0b91d4c1.png">

(add FW NVA routing table)

⚠️ A routing loop has been created.

To understand its origin, we will analyse the packet walk for traffic originated from Spoke1VM towards Branch1 (destination = 192.168.10.1): 
1. As per the ARS programmed entry in Spoke1VM's *Effective routes*, traffic to On-Prem (192.168.0.0/16) is sent to the FW NVA NIC (10.0.0.5)
2. The FW NVA NIC has the same ARS programmed entry, pointing to itself
3. The FW NVA NIC forwards the traffic the FW NVA OS where it is evaluated against the FW NVA routing table
4. The FW NVA routing table has a BGP route for 192.168.0.0/16, Next-Hop = 10.0.10.4 (the Concentrator NVA), and has a route for this Next-Hop pointing to its NIC. 
5. Recursive lookup will result in the traffic being sent back to the FW NVA NIC, that will reforward it up to the FW NVA OS etc 🔁

This is not what we want. We need to reconfigure the FW NVA NIC with a UDR towards the On-Prem branches pointing to the Concentrator NVA NIC (10.0.10.4).

The Concentrator NVA was also in the scope of the ARS and therefore has also a programmed route for the On-Prem branches pointing back at the FW NVA. A similar UDR will be required here as well.

Finally, when looking at the return path (from OnPrem to the Azure VNETs) and as per the Concentrator NVA *Effective routes*, traffic is forwarded based on direct peering connectivity directly to the Spoke VNETs, bypassing the FW NVA, which is not what we want either.

➡️ Introducing ARS with chained NVAs is not as smooth as with a single NVA. BGP propagation of the On-Prem prefixes from the Concentrator NVA to the FW NVA to the ARS is useful for the Spoke VNETs creates issues between the 2 NVAs that need to be corrected. 

### 5.3.1.2. ARS behaviour adjusted with UDRs for OnPrem prefixes

From previous episodes we know now that UDRs would do that perfectly, plus [UDRs are preferred over ARS programmed routes](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#how-azure-selects-a-route): with a UDR configured on the FW NVA towards the On-Prem branch prefixes (192.168.0.0/16) and pointing to the Concentrator NVA (10.0.10.4), the ARS propagated route causing the loop on the FW NVA NIC would get overridden and packets would be forwarded to the Concentrator.

Likewise, the Concentrator NVA has an ARS programmed route for On-Prem prefixes that should be overridden with a UDR to push the traffic up to the Concentrator OS.

### 5.3.1.3. Can ARS influence the Azure VNET ranges?

If you wonder whether ARS could be used to force via the FW NVA the traffic from On-Prem to Azure (by having the FW NVA advertise to the ARS a route towards the VNET ranges) this won't work.

⚠️ ARS cannot be used to reprogram local or peered VNET ranges in the VMs *Effective routes*. Direct VNET peering takes precedence over the ARS route propagation, [even if the ARS propagated routes are more specific](https://learn.microsoft.com/en-us/azure/route-server/route-server-faq#can-i-use-azure-route-server-to-direct-traffic-between-subnets-in-the-same-virtual-network-to-flow-inter-subnet-traffic-through-the-nva).

Again, a UDR matching the Spoke ranges would be required on the FW and Concentrator subnets.

### 5.3.1.4. UDR adjusted connectivity diagram

<img width="1142" alt="image" src="https://user-images.githubusercontent.com/110976272/217070251-f9a6e4d9-eb9a-4b41-91f8-4e3ff43a8bca.png">

The targeted connectivity has been achieved.

The routes programmed by the ARS and overridden by UDRs on both NVA become "*Invalid*".

Let's do the packet walk again:
- Spoke1VM => Branch1 (destination = 192.168.10.1)
    1. As per the ARS programmed entry in Spoke1VM's *Effective routes*, traffic to On-Prem (192.168.0.0/16) is sent to the FW NVA NIC (10.0.0.5)
    2. The FW NVA NIC has now a *User* entry for the branch prefixes pointing to the Concentrator NVA NIC (10.0.10.4)
    3. The Concentrator NVA NIC forwards the traffic to the Concentrator NVA OS where it is evaluated against the Concentrator NVA routing table
    4. Traffic is forwarded to the branches.
- On-Prem => Spoke1VM (destination = 10.1.1.4): the Concentrator NVA routing table directs the traffic towards Spoke1VM to its NIC, containing a *User* entry pointing to the FW NVA NIC, where default VNET peering routing will take the traffic to the Spoke1VM directly.

Compared to the solution proposed in Episode #4, the ARS manages the Spoke VMs routing, limiting the need of UDRs to the FW NVA and the Conczentrator NVA only. However this solution goes against the dynamic routing principles of BGP, requiring to enforce all the VNET ranges and the BGP learnt On-Prem prefixes with UDRs, in numbers that can quickly build up.

## 5.3.2. Chained NVAs, ARS and VxLAN

### 5.3.2.1. Tunneling technique

Instead of implementing UDRs to provide data-plane connectivity between the FW NVA and the Concentrator NVA for all the individual Spoke ranges and On-Prem prefixes, we can also consider using a tunnel between these 2 NVAs and between which there is already default VNET connectivity. BGP will get established inside this tunnel.

Let's go back to the state where the ARS only was deployed, with no UDRs for Spoke or On-Prem connectivity. The routing loop between the FW NVA NIC and the FW NVA OS was caused by the FW NVA NIC being programmed by the ARS to take the 192.168.0.0/16 On-Prem traffic to the FW NVA OS, but then having the FW NVA routing table sending the same 192.168.0.0/16 On-Prem traffic back to its NIC).

When using encapsulation, the On-Prem traffic is hidden inside packets having now the Concentrator tunnel endpoint (10.0.10.4) as destination, not matched anymore by the ARS programmed route (destination = 192.168.0.0/16), and forwarded following direct VNET connectivity to the Concentrator NVA for decapsulation and further On-Prem processing.

In return, the On-Prem traffic destined to the Spokes (destination Spoke1VM = 10.1.1.4) is encapsulated by the Concentrator NVA, sent to its NIC with the destination being now the FW tunnel endpoint (10.0.0.5) for which there is also direct VNET connectivity.

VxLAN encapsulation protocol is used below, but the same can be achieved with IPSec or other tunneling technologies*.

* *Make sure to check the potential performance and/or throughput limitation of the selected tunneling technology.*

<img width="1100" alt="image" src="https://user-images.githubusercontent.com/110976272/217193644-25942e56-1a0e-44c6-a59f-1a0f04da7d92.png">

💡 And finally, end-to-end connectivity has been achieved, dynamically, without any UDR!

# Conclusion & Key Take-aways

I hope this series was of interest to you and helped you have a better understanding of how routing in Azure is done. 

The use cases addressed and solutions provided over the episodes can be combined to provide intermediate or extended scenarios:
- Leveraging ARS for some Spoke VNETs while still keeping UDRs to implement FW bypass for some other Spoke VNETs.
- Keeping it to static routing and UDRs between the 2 NVAs because the On-Prem prefixes and Azure ranges exchanged are too few in perspective of the relative complexity of a VxLAN/IpSec and BGP design or to avoid encapsulation & encryption overhead.
- Providing connectivity between 2 Hub & Spoke environments by deploying an NVA and an ARS like in [this lab](https://github.com/cynthiatreger/double-hub-vnet-and-ars)
- etc

*For a managed version of these deployments you can have a look at [Azure Virtual WAN](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-about).*

And finally, let me share my 3 key take-aways:
- Always make sure your data plane (*Effective routes*) is aligned with the control-plane (NVA routing table)
- Whenever traffic is sent to an NVA, enable "IP forwarding" on the NVA NIC
- Consider the return traffic: it’s not traffic from A to B only, B has to find its way back to A too.

## [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
