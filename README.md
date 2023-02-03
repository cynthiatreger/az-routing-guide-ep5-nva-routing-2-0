<<<<<<< HEAD
### [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
##
# Episode #5: Chained NVAs & BGP

*Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.*
##
[5.1. ccc]()

[5.2. dds]()

&emsp;[4.2.1. vvv]()
##
# recap constraints

the benefits of dynamic routing between the NVAs is plumbed (?) by the heaviness of UDRs required on every subnet + different route tables per envionemnt (Spoke, FW, Concetrator) to steer traffic to the NVA - no central point/device of configuration 

this gets even worse with a variety of On-Prem prefixes, not aggregable under RFC1918 or when public IPs are routed privately (depends on the rest of the customer network etc)

the can quickly get out of control.

simplify implem and operation / deployment, configuration & mgmt

# ARS

## presentation

=> "Before ARS, users had to create a User Defined Route (UDR) to steer traffic to the NVA, and needed to **manually update the routing table on the NVA when VNET addresses got updated**" TO CHECK

ARS 
- enables dynamic routing (BGP) between NVA and local + peered VNETs
unifies the control plane and data plane
With an ARS, OS level routes can be dynamically programmed at the VM NIC level

+ Effective routes are transformed into prefiwes advetised in BGP (ex - VNET ranges advertised by ARS in BGP)

- under the hood iBGP sessions with Virtual Network GWs for further VNG On-Prem to NVA On-Prem interconnectivity

the ARS will trigger route propagation in its local VNET and the peered VNETS and will be subject
The GW Transit anf GW route propagation settings do apply to ARS.
ARS route propagation will be subject to the 
The ARS route propagation impacts the same scope than a Virtula Network GW (local + peered VNETs)
follows the same 

ARS not in the data-path, works like a RR
The ARS is NOT in the data path but will enable the routing information received from the BGP peering between the ARS and the CSR NVA to be added to the Hub and peered VNETs.

ARS in the RouteServerSubnet

check out Mays' [ARS MH](https://github.com/malgebary/Azure-Route-Server-MicroHack) covering commmon routing scenarios leveraging ARS to understand/get its full power "to simplify configuration, management, and deployment of the NVAs in the Virtual Network and how would that simplify routing within Azure and between Azure and on-premises."

[Adam's video on ARS placement](https://youtu.be/eKRuJPjCR7o)

## Ep 3 topology and ARS (simple NVA environment)


to illustrate the impact and power of ARS, we will start with an Ep3 like environment
- 1 hub VNET peered with 2 spokes 
    - GW transit enabled for Spoke1
    - GW route propagation disabled on Spoke1/subnet2 (Spoke1VM2)
    - GW disabled on the Spoke2 peering (Spoke2VM)
- Concentrator NVA for brnach connectivity
- all the UDRs configured in the previous episodes have been removed, 
- the 10/8 static route configured on the Concentrator and advertised to On-Prem removed

=> Azure rerachability from the branches was enabled by a static 10/8 route advertised by the Concentrator to the OnPrem
Episode 3: The static 10/8 supernet route covering the Azure environment and pointing to the subnet default gateway (10.0.10.1) is configured on the Concentrator NVA to be further advertised to the branches.
=> not needed anymore

ARS added in Ep3 setup, in which all the UDRs have been removed.

ARS BGP peered with Concentrator NVA

**IMAGE 1**

with just ARS + BGP with Concentrator, the whole Episode 3 lab has been completed, the On-Prem reachability extended to the ARS VNET + the peered VNETs just like in VNG use case
without any UDRs

Gw transit and gw rout propagation are honored for NVA advertised prefixes as shown by the Eff routes of Spoke1VM2 and Spoke2VM + failed pings

We have demonstrated GW route prop and GW Transit is honoured. For the rest of this episode, we will focus on Spoke1 only (GW Transit + GW route prop = ON)
## Chained NVAs and ARS

ARS peer with FW?

FW advertises to the ARS the On-Prem branches
makes sense to receive thiese On-Prem branches forn the Cpncentrator via BGP too.

and then ARS programs them in the VM Effective routes
all the spokes will know onprem reachable via FW

but:

- Episode 4 demonstrated that valid routing tables doesn't mean valid data planes connectivity, so although BGP makes sense for route advertisement to the ARS and further propagation to the VMs in the local + peered VNETs UDRs would again still be needed to align the data plane
makes sense to also have bgp running from the NVA to the concentrator
no data-plane connectivity from FW to Concetrator, so back to AGAIN, use UDRs.

-Alternatively, could ARS be used to infuence the return trffic? can , via route to VNET advertised by FW to ARS, ARS help force the On-prem traffic received on the cicnentrztor to be routed to the NVA?
No
https://learn.microsoft.com/en-us/azure/route-server/route-server-faq#can-i-use-azure-route-server-to-direct-traffic-between-subnets-in-the-same-virtual-network-to-flow-inter-subnet-traffic-through-the-nva

### try to push the Spoke VNET ranges to the NVA Effective routes?
 won't work as VNET ranges  learnt by direct VNET peering, takes precedence anyway over the ARS propagated routes.

so again, UDRs would be needed.

Unless...

## tunneling techniques

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

video it's the end

## [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
