<<<<<<< HEAD
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

In the 2 last episodes we enabled connectivity end-to-end between Azure and branches connected to a non-Azure-native Concentrator ([Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals)) and then added FW inspection via a non-Azure-native FW ([Episode #4](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas)).

We demonstrated that OS-level routing was not enough and that the NIC *Effective routes* of all the VMs in the local and peered VNETs had to be aligned using UDRs. These UDRs had to be applied on every subnet, using separate *Route tables* per environment (Spoke, FW NVA, Concentrator NVA), without any central point of configuration.

For larger deployments with tens or even a few hundreds Spoke ranges and a variety of On-Prem prefixes, not always strictly aggregable under RFC1918, this can quickly become complex to both implement and operate.
##
# 5.1. Benefits of Azure Route Server (ARS)

the [Azure Route Server](https://learn.microsoft.com/en-us/azure/route-server/overview) (ARS) is a Azure resource deployed in a dedicated subnet (that must be name *RouteServerSubnet*) that acts like a Route Reflector and whos job is basically to align the control plane (NVA routing table) and the data-plane (*Effective routes*).

When deployed, the ARS can run BGP with an NVA and as a result will dynamically propagate and program the routing information received from this NVA in the *Effective routes* of all the VMs in the local and peered VNETs.
The ARS also works the other way round and advertises the local and peered VNET ranges to its BGP-peered NVA.

When the ARS is deployed in a VNET with an Azure Virtual Network Gateway, iBGP is automatically run between them and can be enabled to provide branch-to-branch transit.

The ARS takes the control over from the Azure Virtual Network Gateways if any, and becomes the new controler of the Azure scope. Consequently, the *GW Transit* and *GW route propagation* settings apply to ARS.

⚠️ Please note the ARS is NOT in the data-path. For non-Azure-native scenarios, the ARS must be paired with an NVA to manage the traffic.

In this episode we are going to address one of the many solutions ARS provides. If you would like to find out more about the routing scenarios leveraging ARS, please check out Mays' [ARS MicroHack](https://github.com/malgebary/Azure-Route-Server-MicroHack). 

Adam's [video on ARS placement](https://youtu.be/eKRuJPjCR7o) has also been a great support.

# 5.2. Episode #3 topology (single NVA) and ARS

To illustrate the impact and power of ARS, we will start with an Episode #3 like environment, updated with an ARS: 
- 1 Hub VNET peered with 2 Spokes (Spoke1 and Spoke2)
    - *GW transit* enabled for Spoke1
    - *GW route propagation* disabled on Spoke1/subnet2
    - *GW transit* disabled on the Spoke2 peering
- 1 Concentrator NVA for On-Prem branch connectivity
- 1 ARS peered with Concentrator NVA

*All the *Route tables* and UDRs configured in the previous episodes have been removed, as well as the 10/8 static route configured on the Concentrator and advertised On-Prem for Azure reachability.*

**IMAGE 1**

➡️ automatic route propagation and programmation both at the NC level and os level
algines the effectives routes of the VMs in the ARS VNET and peered VNETs to the Conc NVA routing table.

with just ARS + BGP with Concentrator, the whole Episode 3 lab has been completed, the On-Prem reachability extended to the ARS VNET + the peered VNETs just like in VNG use case
without any UDRs

Gw transit and gw rout propagation are honored for NVA advertised prefixes as shown by the Eff routes of Spoke1VM2 and Spoke2VM + failed pings

We have demonstrated GW route prop and GW Transit is honoured. For the rest of this episode, we will focus on Spoke1/subnet1 only (GW Transit + GW route prop = ON)

# 5.3. Episode #4 topology (chained NVAs) and ARS

# recap episode 4 & constraints

In the previous [episode] we enabled FW inspection and connectivity end-to-end between Azure and branches connected to a Concentrator. 

Although the routing between the FW NVA and the Concentrator NVA was well implemented (static routes or BGP), we demonstrated that OS-level valid routing was not enough, providing control-plane connectivity only. 

The NIC *Effective routes* of all the VMs in the local and peered VNETs had to be aligned to the NVA routing tables to steer traffic through the FW and finally achieve data-plane connectivity:
- UDRs configueed on every subnet
- different *Route tables* per environemnt (Spoke, FW NVA, Concentrator NVA) to steer traffic to the NVA - no central point/device of configuration 

if 2 peerings - ECMP
ARS peer with FW?

FW advertises to the ARS the On-Prem branches
makes sense to receive thiese On-Prem branches forn the Cpncentrator via BGP too.

and then ARS programs them in the VM Effective routes so the spokes  will know onprem reachable via FW

### ARS behaviour 

### On-Prem prefixes

**IMAGE 2**

packet walk

warning trceroute => loop
the ARS programs the Effective routes of the FW NVA with a route for the branches to itself: packets managed by NVA rouitng tablr with NH = Concentrator , recursive lookup sends to FW NIC, and here FW NIC sends up to the FW OS
 etc

This is not what we want.
need to reprogram the FW NIC with a route towards branches pointing to the Cpncentrator 

btw, need the same route on the concentrator because at the moment concentraote also has branches reachable through FW in its Effective routes

Concentrator/FW BGP tp push to the ARS the On-Prem routzs dynamically but then ARS floods its local and peered vnets with these routes and programs them in the VM NIC's Effective routes, with NH being its BGP peered NVA. 

finally, return path (OnPrem => Az): according to the Effective routes of Conc NVA, direct forward to the Spoke1VNET, no FW transit, so that needs to be adjusted too

introdcing ARS with chained NVAs, the  BGP advertised On-Prem prefixes from the Conc to the FW to the ARS is useful for the spokes but creates issues in the hub VNET that need to be corrected

### ARS behaviour adjusted with UDRs for OnPrem prefixes
ARS's behaviour needs to be contained:
From episodes we know now that UDRs would do that perfectly: with a UDR towards the On-Prem branch prefixes and pointing to the Concentrator NVA configured on the FW NVA, the ARS propagated route causing the loop on the FW NVA NIC would get overriden and packets would be forwarded to the Concentrator.

But likewise, the Concentrator at the moment has in its Effective routes an entry for the On-Prem branches programmed by the ARS with NH = FW NVA. Here too a UDR is required to force the traffic up the NIC to the Concentrator OS.


### Az ranges? try to push the Spoke VNET ranges to the NVA Effective routes?
foreign prefixes?
use of ARS to force via FW to infuence the return trffic? can , via route to VNET advertised by FW to ARS, ARS help force the On-prem traffic received on the cicnentrztor to be routed to the NVA?
static route FW for spoke subnet pointing to itself
No
https://learn.microsoft.com/en-us/azure/route-server/route-server-faq#can-i-use-azure-route-server-to-direct-traffic-between-subnets-in-the-same-virtual-network-to-flow-inter-subnet-traffic-through-the-nva

 won't work as VNET ranges learnt by direct VNET peering, takes precedence anyway over the ARS propagated routes.
 if longer prefix?

so again, a UDR matching the Spoke ranges (no supernet because of LPM) on the Concentrator subnet would be needed


**IMAGE 3**

routes advertised rom ARS become invalid etc


### Packet walk:
Spoke1=>OnPrem (follow the 192.68.0./16
Spoke1VM NIC (Eff route: ARS programmed entry) => FW NVA NIC (EFF routes: UDR entry) => Conc NVA NIC (UDR entry) => Conc NVA OS (routing table) => Branches
- traffic destination matched against Spoke1VM's NIC Eff R => ARS programmed a 192.168/16 routes to FW, packets send over Az platform to FW NIC
- FW NIC Eff routes contain a UDR for the branch prefixes so packet forwarded to NH specified = Conc
- Packets reach the con NIC, eff route with UDR to its own IP, packets get processed by the OS, routing table consulted: traffic forwarded over the tunnels to the branches (IPSEc or SDWAN whatver)

OnPrem=>Spoke (we rtack the Spoke1VM subnet)
- Conc OS (routing table) => Conc NIC (UDR) => FW NIC => Spoke1VM NIC

### conclusion
!it worksn, reduces the need of UDRs to the Hub VNET only, but goes against the dynamic routing concepts of BGP, requiring again to enforce all the foreign dynamically leanrt routes with static routes at the NIC level...

## tunneling techniques

The ARS steers traffic to its peered NVA. In the case of chained NVAs, there is an additional hop to consider that consists of pushing that steered traffic one hop further, from the FW NVA to the Conc NVA. 
UDRs are one way to do it as they enable the reachability over this addiitonal hop.
Although ARS here helped limiting the need of UDRs to only the FW and the Conc
For larger deployments with 10s or even a few hundred Spoke ranges, and On-Prem prefixes that cannot always be aggregated easily to RFC1918, this can again quickly become out of control

UDRs to override every on-prem prefixe programmed by ARS

Another way to address routing over this addiitonal hop is to use tunneling between the chained NVAs, like IPSec or VxLAN, 
The FW NVA would encapsulate traffic from the spokes to the Branches befofe sending them to the Conc.
The Conc would encapsulate traffic from the Branches to the Spokes before sending them to the FW.

Instead of ensuring data-plane connectivity for all the individual spoke ranges and On-Prem prefixes between the FW NVA and the Cn NVA, we would just have to establish data-plane connecitivty between the 2 tunnel endpoints on each NVA (1 UDR)

whatever branch dest or az src always encaps in same source dest (tunnel)

 
Let's go back to ARS only prgrammed routes, and the loop on the FW NIC caused by the NIC being programmed by ARS to push the OnPrem traffic up to its OS, but then having the FW routing table matching the On-Prem dest + recursive lookup on NH = the Conc + sending it back down to the NIC.

what if NH = tunnel dest
tunnel dest 

once attracted to the ARS peered NVA, the UDRs configured  

the need of UDRs here comes from the fact that the NH programmed by the ARS is not the last NH for the dest prefixes in the data plane

ARS propagates prefixes received over BGP from its neighbors (here FW NVA) to its local VBNET and the peered VNETs and programs the Eff routes of each VM with the destination prefixes, NH = its BGP neighbor

Eff routes programmation done by ARS for the propagated prefixes
results in traffic the traffic to the peered NVA

unless we use encaps technique to trick the azure platform
like IPSec or VxLAN create a tunnel between 2 routing devices, 2 endpoints in the local VNET (direct connectivity between them on the data plane), here our NVAs.
having a tunnel to avoid the 

and then BGP run within the tunnel
///
 ### BGP between the NVA and the Cocnentrator

 Let's now focus on the BGP route exchanges between the FW NVA and the Concentrator NVA:

- FW NVA:
    - advertises via BGP the specific Spoke1 VNET range (10.1.0.0/16) to the Concentrator NVA
    - learns via BGP the On-Prem branch prefixes supernet (192.168.0.0/16) from the Concentrator NVA: Next-Hop = 10.0.10.4

- Concentrator NVA:
    - advertises via BGP the OnPrem branch prefixes supernet (192.168.0.0/16) to the FW NVA
    - learns the Spoke1 VNET range and FW NVA range via BGP from the FW NVA: Next-Hop = 10.0.0.5


## 
intermediate scenario,
ARS for some routes
UDRs for others, for example for mixing FW inspection and bypass depending on the spokes
# Key take aways

routing table and Effective routes alignment

Consider return traffic
It’s not traffic from A to B only, B has to find its way back to A too.

Hide in the tunnel 

probably about ARS propagating VNET subnets not taking precedence on default VNET routing

simplify implem and operation / deployment, configuration & mgmt


=> "Before ARS, users had to create a User Defined Route (UDR) to steer traffic to the NVA, and needed to **manually update the routing table on the NVA when VNET addresses got updated**" TO CHECK

video it's the end

## [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
