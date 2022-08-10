# High availibility is not the same as disaster recovery

## Intro:

This example scenarios are applicable to any industry that needs to deploy resilient Azure networking built for high availability and disaster recovery. In these scenarios, we look at a two region logical architecture layout for a classic Azure virtual network Hub & Spoke, with Express Route, S2S VPN, Azure Firewall and Azure Front Door.

When designing your Azure network to be resilient, you must understand your availability and disaster recovery requirements. How much downtime is acceptable? The amount of downtime is partly a function of cost. How much will potential downtime cost your business? How much should you invest in making the Network highly available and resilient to failuer?

High availability and disaster recovery follow some of the same best practices but with different strategies being applied at all levels of the architecture. Some mitigations are more tactical and others more strategic in nature for example:

- Retrying a remote call after a transient network failure.
- Automatically failing over the entire network to a secondary region.
- Having the right monitoring and diagnostics to to detect failures when they happen, and to find the root causes

It's rare for an entire region to experience a disruption, but transient problems such as network congestion are more common so target these issues first. With that said your availability will focus on components of the network, and disaster recovery will focus on discrete copies of the entire network. Hence DR (Disaster Recovery) has different objectives from HA (High Availability). So your DR design and strategy will require a different apporach than those for HA. DR is focused on deploying individually separate and distinct network components to multiple locations, so that you can fail over the entire network if necessary to a different region ensuring your network meets your availability objectives, and commitments you make to your customers. 

Architecting HA and DR into your network ensures your workloads are available and can recover from failures at any scale. 

## Architecture diagrams

## Scenario 1
### HA & DR Virtual Network Hub & Spoke Express Route, Azure Firewall and Azure Front Door

![image](https://user-images.githubusercontent.com/81341827/183797963-a862c0ee-4533-49d7-aad5-fdb08fbe3461.png)

## Scenario 2
### HA & DR Virtual Network Hub & Spoke Azure Express Route, S2S VPN, Azure Firewall and Azure Front Door

![image](https://user-images.githubusercontent.com/81341827/183798668-7cd4d1f6-8eb1-4f57-a778-8e688f669f30.png)


## Architecture components, configuration designs and considerations

### Chose the right primary and secondary Azure regions

The best practice is to select regions that are closer to your on-premises locations since the distance the data has to travel will have a direct impact on your overall network latency. Use two regions to achieve DR one is the primary region. The other region is the failover secondary region.  

### Resource groups

Create separate resource groups for the primary region and secondary region. This gives you the flexibility to manage each region as a single collection of resources. For example, you could redeploy one region, without taking down the other one.

### Virtual Networks

Create a separate virtual network for each region. Make sure the address spaces do not overlap.

### Regional HA | Availability Zone aware ExpressRoute and S2S VPN virtual network gateways

An Availability Zone in an Azure region is a combination of a fault domains and an update domains. Opt for zone-redundant virtual network gateways that terminate ExpressRoute private peering and S2S VPN connection. Configure zone-redundant virtual network gateway in both your primary and secondary regions.

For high availability, it's essential to maintain the redundancy of the ExpressRoute circuit and S2S VPN throughout the end-to-end network. In other words, you need to maintain redundancy within your on-premises network, and shouldn't compromise redundancy within your service provider network. Maintaining redundancy at the minimum implies avoiding single point of network failures. Having redundant power and cooling for the network devices will further improve the high availability.

### Configure a Site-to-Site VPN as a failover path for ExpressRoute
You can configure a Site-to-Site VPN connection as a backup for ExpressRoute this is shown in the #scenario 2 diagram. This connection applies only to virtual networks linked to the Azure private peering path. There's no VPN-based failover solution for services accessible through Azure Microsoft peering. The ExpressRoute circuit is always the primary link. Data flows through the Site-to-Site VPN path only if the ExpressRoute circuit fails. To avoid asymmetrical routing, your local network configuration should also prefer the ExpressRoute circuit over the Site-to-Site VPN. You can prefer the ExpressRoute path by setting higher local preference for the routes received the ExpressRoute. 

### Regional HA - Azure Firewall

Azure Firewall is a cloud-native and intelligent network firewall security service that provides the best of breed threat protection for your cloud workloads running in Azure. It's a fully stateful, firewall as a service with built-in high availability and unrestricted cloud scalability. It provides both east-west and north-south traffic inspection using rules that define allowed and denied network traffic. 

Use Azure Firewall Manager to centrally manage Azure Firewalls across multiple subscriptions in different regions. Firewall Manager leverages firewall policy to apply a common set of network/application rules and configuration to the firewalls in your tenant. 


### BCDR - Multi Region Failover

I have already called out that DR is focused on deploying individually separate and distinct network components to multiple locations, so that you can fail over the entire network if necessary. In order safeguard against disasters that impact an entire peering location or IPsec connection, your disaster recovery plans should include geo-redundant ExpressRoute circuits, S2S IPsec tunnels or a mixture of both, this is shown in the logical layouts for both scenarios #1 and  #2

ExpressRoute supports Bidirectional Forwarding Detection (BFD) both over private and Microsoft peering. When you enable BFD over ExpressRoute, you can speed up the link failure detection between Microsoft Enterprise edge (MSEE) devices and the routers that your ExpressRoute circuit gets configured (CE/PE). This document walks you through the need for BFD, and how to enable BFD over ExpressRoute. [https://docs.microsoft.com/en-us/azure/expressroute/expressroute-bfd]

### Azure Front Door 

Front Door Is an application delivery network that provides global load balancing and site acceleration service for web applications. It offers Layer 7 capabilities for your application like SSL offload, path-based routing, fast failover, caching, etc. Use Azure Front Door to provide active/passive or active/active fail over scenarios for your applications if a regional outage affects the primary region, you can use Front Door to fail over to the secondary region.

## Next steps

To learn more about Azure networking HA and DR see:

- https://docs.microsoft.com/en-us/azure/expressroute/designing-for-high-availability-with-expressroute
- https://docs.microsoft.com/en-us/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering
- https://docs.microsoft.com/en-us/azure/expressroute/use-s2s-vpn-as-backup-for-expressroute-privatepeering
- https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-activeactive-rm-powershell
- https://docs.microsoft.com/en-us/azure/firewall-manager/policy-overview




