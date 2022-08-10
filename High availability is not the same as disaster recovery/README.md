# High availibility is not the same as disaster recovery

## Intro:

This example scenarios are applicable to any industry that needs to deploy resilient Azure networking built for high availability and disaster recovery. In this scenario, we look at a two region logical architecture layout for a classic Azure virtual network Hub & Spoke and Azure Front Door.

When designing your Azure network to be resilient, you must understand your availability and disaster recovery requirements. How much downtime is acceptable? The amount of downtime is partly a function of cost. How much will potential downtime cost your business? How much should you invest in making the Network highly available and resilient to failuer?

High availability and disaster recovery follow some of the same best practices but with different strategies being applied at all levels of the architecture. Some mitigations are more tactical and others more strategic in nature for example:

- Retrying a remote call after a transient network failure.
- Automatically failing over the entire network to a secondary region.
- Having the right monitoring and diagnostics to to detect failures when they happen, and to find the root causes

It's rare for an entire region to experience a disruption, but transient problems such as network congestion are more common so target these issues first. With that said your availability will focus on components of the network, and disaster recovery will focus on discrete copies of the entire network. Hence DR (Disaster Recovery) has different objectives from HA (High Availability). So your DR design and strategy will require a different apporach than those for HA. DR is focused on deploying individually separate and distinct network components to multiple locations, so that you can fail over the entire network if necessary to a different region ensuring your network meets your availability objectives, and commitments you make to your customers. 

Architecting HA and DR into your network ensures your workloads are available and can recover from failures at any scale. 

## Architecture

## HA & DR Virtual Network Hub & Spoke Express Route With and Azure Front Door

![image](https://user-images.githubusercontent.com/81341827/183797963-a862c0ee-4533-49d7-aad5-fdb08fbe3461.png)


## HA & DR Virtual Network Hub & Spoke Azure Express Route, S2S VPN and Azure Front Door

![image](https://user-images.githubusercontent.com/81341827/183798668-7cd4d1f6-8eb1-4f57-a778-8e688f669f30.png)


## Components

## Chose the right primary and secondary Azure regions

Agree on your primary and secondary regions the best practice is to select regions that are closer to your on-premises locations since the distance the data has to travel will have a direct impact on your overall network latency.

## Regional HA | Availability Zone aware ExpressRoute and S2S VPN virtual network gateways

An Availability Zone in an Azure region is a combination of a fault domain and an update domain. If you opt for zone-redundant Azure IaaS deployment, you may also want to configure zone-redundant virtual network gateways that terminate ExpressRoute private peering and S2S VPN connection. Configure zone-redundant virtual network gateway in both your primary and secondary regions. 

For high availability, it's essential to maintain the redundancy of the ExpressRoute circuit and S2S VPN throughout the end-to-end network. In other words, you need to maintain redundancy within your on-premises network, and shouldn't compromise redundancy within your service provider network. Maintaining redundancy at the minimum implies avoiding single point of network failures. Having redundant power and cooling for the network devices will further improve the high availability.

## Azure Front Door 

Front Door Is an application delivery network that provides global load balancing and site acceleration service for web applications. It offers Layer 7 capabilities for your application like SSL offload, path-based routing, fast failover, caching, etc. Use Azure Front Door to provide active/passive or active/active fail over scenarios for your applications if a regional outage affects the primary region, you can use Front Door to fail over to the secondary region.


