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

- under the hood iBGP sessions with Virtual Network GWs for further VNG On-Prem to NVA On-Prem interconnectivity

check out Mays' [ARS MH](https://github.com/malgebary/Azure-Route-Server-MicroHack) covering commmon routing scenarios leveraging ARS to understand/get its full power "to simplify configuration, management, and deployment of the NVAs in the Virtual Network and how would that simplify routing within Azure and between Azure and on-premises."

## Ep 3 topology and ARS

to illustrate the impact and power of ARS, ARS added in Ep3 setup, in which all the UDRs have been removed.

ARS BGP peered with Concentrator NVA

### Gw transit and gw rout propagation


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

# Key take aways

routing table and Effective routes alignment

Consider return traffic
It’s not traffic from A to B only, B has to find its way back to A too.

Hide in the tunnel 

probably about ARS propagating VNET subnets not taking precedence on default VNET routing
