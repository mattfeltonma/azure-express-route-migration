# Azure ExpressRoute Migration
This repository contains a migration guide that could be used for migrations between ExpressRoute circuits. This guide builds off of [Adam Stuart's excellent repository](https://github.com/adstuart/azure-expressroute-migration) on the topic.

## Decisions to be made
### General
* Customer validates that the subscription they are creating the ExpressRoute Circuit in has sufficient quota for the ExpressRoute Circuit [(50 total, 10 per region)](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#expressroute-limits)
* Customer validates they do not already have 4 ExpressRoute Virtual Network Connections to the ExpressRoute Gateway from the same peering location [(this is the maximum)](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways#gatewayfeaturesupport)
* Customer validates that the ExpressRoute Gateway SKU they plan on using [meets their availability and throughput requirements](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways#gatewayfeaturesupport)
* Customer determines what the size of the circuit will be and whether the SKU will be local, standard, or premium
### Networking
* Customer reserves new VLAN tags for each new circuit ExpressRoute Private Peering.
* [Customer reserves one IPv4 /30 subnet for primary link and one IPv4 /30 for the secondary link per circuit being created](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-routing-portal-resource-manager#to-create-azure-private-peering)
* [Customer reserves an AS number for Private Peering for each circuit](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-routing-portal-resource-manager#to-create-azure-private-peering)
* Customer reserves IP space for the newly connected virtual networks that does not overlap with existing IP space on-premises or in Azure

## Implementation
### Implementation Phase 1
1. Create new circuit with chosen bandwidth and SKU
2. Create private peering
3. (If different subscription or tenant) Record the ExpressRoute Circuit resource id
4. (If different subscription or tenant) [Create a new authorization key for the ExpressRoute Circuit](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-portal-resource-manager?pivots=expressroute-current#circuit-owner-operations). You will need two authorization keys if you plan to do the testing in Testing Phase 1. Each authorization key is one time use and allows one ExpressRoute Gateway to be connected to the ExpressRoute Circuit.

### Testing Phase 1
1. Create a new virtual network and ExpressRoute Gateway in the destination Azure subscription
2. [Create a new virtual network connection to the new virtual network and ExpressRoute Gateway created in the prior step]([Link a virtual network to ExpressRoute circuits - Azure portal | Microsoft Learn](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-portal-resource-manager?pivots=expressroute-current#to-create-a-connection))
3. Create a [new connection from the ExpressRoute Circuit to the ExpressRoute Gateway in the new virtual network](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-cli#connect-a-virtual-network-in-the-same-subscription-to-a-circuit). If the ExpressRoute Gateway and Virtual Network are in a different Entra ID you must create a new connection and [provide the authorization key and peer circuit resource id]([Link a virtual network to ExpressRoute circuits - Azure portal | Microsoft Learn](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-portal-resource-manager?pivots=expressroute-current#circuit-user-operations)) 
4. Validate that the new Azure IP space you configured on the connected virtual network is being received on your routers
5. [Validate that the new Azure IP space is present and the IP space you are advertising on-premises appears in the route table of the ExpressRoute private peering](https://blog.cloudtrooper.net/2021/07/12/cli-based-analysis-of-an-expressroute-private-peering/)
6. Create a new virtual machine in the new virtual network with the ExpressRoute Gateway and validate that you can connect to the virtual machine.
7. Delete the virtual network, ExpressRoute Gateway, and virtual machine you used for this testing phase.

### Implementation Phase 2 (Active/Passive setup with current circuit as primary and new circuit as secondary)
1. [Control traffic from Azure to on-premises such that Azure will prefer the existing ExpressRoute circuit connection. Do this by setting the weight on the existing connection to 100 (higher gets preference) or by AS-PATH prepending the routes you are advertising to Azure on the BGP connection on the new circuit.](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-optimize-routing)
2. Control traffic from on-premises to Azure such that on-premises will prefer the existing ExpressRoute circuit connection. Do this by tuning BGP metrics. You could tune local preference or use a route map on your router to AS-PATH prepend the routes you receive from the new ExpressRoute circuit.
3. Connect the new ExpressRoute circuit to the existing ExpressRoute Gateway that exists in the virtual network you are migrating to the new circuit. Ensure you set the weight as 0 You can use step 2-3 from Testing Phase 1. This creates an active/passive setup where the current ExpressRoute circuit is active and the new ExpressRoute circuit is passive.

### Testing Phase 2
* Repeat steps 4-5 from Testing Phase 1 to validate the routes are being exchanged between your router and the new BGP peers that are part of the Private Peering configuration on the new ExpressRoute Circuit.

### Implementation Phase 3 (Change new circuit to active and existing circuit to passive)
You have two options to perform the cutover.
#### Option 1 (Less Downtime) May result in temporary traffic asymmetry depending on BGP propagation timing and whether firewall is positioned in front of or behind customer router
  1. (Traffic from Azure to On-Premises) [Modify the weight of the ExpressRoute Gateway connection](https://learn.microsoft.com/en-us/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering#connection-weight) to the new circuit to a weight of 200 (if using AS-PATH Prepending on-premises you will need to make changes there to make the new circuit preferred)
  2. (Traffic from On-Premises to Azure) Modify the BGP metrics you configured in implementation phase to so that traffic from on-premises to Azure will prefer the new circuit.
  3. Delete old connection and get rid of old circuits
#### Option 2 (10s (BFD)- 240s downtime (no BFD))
  1. Delete the connection object that links your old ExpressRoute circuit to the ExpressRoute Gateway. This will result in some downtime. It is typically 10 seconds if using BFD (Bidirectional Forwarding Detection) or up to 240 seconds without BFD.

### Testing Phase 3
1. Repeat steps 4-5 from Testing Phase 1 to validate the routes are being exchanged between your router and the new BGP peers that are part of the Private Peering configuration on the new ExpressRoute Circuit.
2. Perform some type of connectivity testing to existing workloads connected to the virtual network containing the ExpressRoute Gateway.

## Helpful Links
* [Architecting ExpressRoute for Disaster Recovery](https://learn.microsoft.com/en-us/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering)
* [Adam Stuart's Repository on ExpressRoute Migration](https://github.com/adstuart/azure-expressroute-migration)
