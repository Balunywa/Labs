
Forced tunneling lets you redirect or “force” all Internet-bound traffic back to your on-premises location via a Site-to-Site VPN or Express Route for inspection and auditing.

This is a critical security requirement for most enterprise IT policies. If you don’t configure forced tunneling, Internet-bound traffic from your VMs in Azure always traverses from the Azure network infrastructure directly out to the Internet, without the option to allow you to inspect or audit the traffic.

There are scenarios when customers require to have both internet breakout from Azure and forced tunneling.

With Site-to-Site VPN redirecting traffic is expressed by mapping a UDR (User Defined Route) to your routing tables with the next hop as the virtual network gateway and associating the RTB (Routing Table) to your specific subnets.

However, with Express Route Gateways you must use BGP to advertise on-premises routes to the Microsoft Edge router. You can’t create user-defined routes to force traffic to the ExpressRoute virtual network gateway if you deploy a virtual network gateway deployed as type ExpressRoute.

You can use user-defined routes for forcing traffic from the Express Route to, for example, a Network Virtual Appliance.

This article looks at how to accomplish forced tunnelling and internet breakout via Azure at the same time in scenarios when Express Route is being used and a default route 0/0 is being advertised via BGP.

## Scenario

Contoso has a classic hub and spoke network with express route for private connectivity back to their on-premises, and currently all internet breakout traffic from Azure was going through set for Firewall configured with HA (High Availability) in Azure, however, their IT security team has come up with a requirement that mandates certain type of workloads to be forced tunneled through their Express Route Circuits back to their on-premises for further inspections and auditing

## Requirements

# Spoke1 vNet

Workloads should have network line of site to on-premises resources through an express route connection and internet breakout should go over NVA’s (Network Virtual Appliances) that are deployed within the Hub vNet in Azure to get traffic inspected

# Spoke2 vNet

Workloads breakout should be forced tunneled through Express Route and internet breakout should only happen on their on-premises firewalls.

# The following diagram illustrates these two-traffic flow requirements.

![image](https://user-images.githubusercontent.com/81341827/187049985-e406e73b-ede5-4a68-9f8b-8c1a834bbdc1.png)



# Solution and considerations

Forced tunneling in Express Route is enabled by advertising a default route 0/0 via the ExpressRoute BGP peering sessions.

It’s important to note that if multiple routes contain the same address prefix, Azure will select the route type, based on the following priority:

User-defined route
BGP route
System route

# Hub vNet:

Create the Hub-vNet with a Gateway Subnet (deploy ER-GW in this subnet) and an NVA Subnet (deploy the NVA’s in this subnet)

Once you link ER-Gateway to the Express Circuit then BGP will inject 0/0 in your NVA Subnet with the next hop as the ER-Gateway, in order for the NVA to have internet break out you have to create a UDR with 0/0 with the Internet as the next hop. (This will override the 0/0 injected by ER-GW via BGP and prevent the NVA from force tunneling internet breakout traffic to on-premises)

# Spoke1 vNet:

Create the Spoke1 vNet with vNet peering to the Hub vNet and enable the use Remote Gateway flag.

This will inject all routes learned via BGP into the Spoke1 Subnets including the 0/0 default route. In order for Spoke1 to have internet break out via the NVA another UDR with 0/0 default route needs to be created with next hop as the NVA’s private ip address prefix. hence this will be overriding the 0/0 default route injected by the ER-GW via BGP and push all internet bound traffic via the NVA.
Important to note that if you have any Azure PaaS services deployed in the Spoke1 vNet they will need to have network line of site to the MSFT backbone to access the control plane in order to function as expected and 0/0 default route will break that control plane, hence the need create UDR Tags (Group of IP’s fully managed by MSFT) for those PaaS services, this can be associated Spoke1 Subnets or at the NVA Subnets. Once associated control plane traffic will go to the MSFT backbone.

# Spoke2 vNet:

Create the Spoke2 vNet with vNet peering to the Hub vNet and enable the use Remote Gateway flag.

This will inject all routes learned via BGP into the Spoke2 Subnets including the 0/0 default route. By default, this will force tunnel all internet bound traffic back to on-premises via the Express Route Gateway which meets the security teams requirements for Spoke2. Hence, intentionally NOT adding a UDR with 0/0 to the NVA’s as the next hop.
Important to note that if you have any Azure PaaS services deployed in the Spoke2 vNet they will need to have network line of site to the MSFT backbone to access the control plane in order to function as expected and 0/0 default route will break that control plane, hence the need create UDR Tags (Group of IP’s fully managed by MSFT) for those PaaS services associated Spoke2 Subnets

# MSEE (MSFT Enterprise Edge Routers) Spoke vNet’s Hair pinning

Once 0/0 is advertised via ER-BGP or summary routes for example 10.0.0.0/8, these routes are injected into all the Spoke vNets peered to the Hub vNet, which will introduce transit routing, (all your Spoke vNet’s directly peered to the Hub vNet will have network line of site to each other) through hair pinning the MSEE routers. To avoid transit routing through the MSEE you need to ensure that you don’t propagate from ER-BGP on-premises summary that incorporates your Azure vNet Spoke prefixes. So, for example if you have class A routes (10/24, 10/16, 10/20..etc) in Azure then don’t propagate 10.0.0.0/8 via ER-BGP from on-premises since that will hairpin through the MSEE and enable transit across all your vNets.

# Next steps
To learn more about how Azure routes virtual network traffic see:

https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview

https://docs.microsoft.com/en-us/azure/virtual-network/service-tags-overview

