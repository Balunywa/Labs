# Forced Tunneling and Internet Breakout with Express Route & NVA Scenario

Forced tunneling lets you redirect or "force" all Internet-bound traffic back to your on-premises location via a Site-to-Site VPN or Express Route tunnel for inspection and auditing. This is a critical security requirement for most enterprise IT policies. If you don't configure forced tunneling, Internet-bound traffic from your VMs in Azure always traverses from the Azure network infrastructure directly out to the Internet, without the option to allow you to inspect or audit the traffic. 

There scenraios when customers requires to have both internete breakout from Azure and forced tunneling. With S2S VPN you can do this by mapping a UDR (Userd defined route) to your subnets with the next hop as the the virtual network gateway. However, with Express Route Gateways you must use BGP to advertise on-premises routes to the Microsoft Edge router. You can't create user-defined routes to force traffic to the ExpressRoute virtual network gateway if you deploy a virtual network gateway deployed as type: ExpressRoute. You can use user-defined routes for forcing traffic from the Express Route to, for example, a Network Virtual Appliance.

These article looks at how to acomplish forced tunelling and internet break in scenarios when Express Route is being used and a defualt route 0/0 is being adversitised via ER BGP.

## Scenario

Contoso has a classic hub and spoke netwok with express route for private connectivity back to their on-premises, currently all internet breakout traffic from Azure was going through set for Firewall configured with HA (High Aviablility) in Azure, however, their IT secuirty team has come up with a requirement that manadates certian type of worksloads to be forced tunneled through their Express Gateways back to their on-prmeises for further inspections and auditing before internet breakout.

Requirements
Spoke 1 workloads Networking Breakout should go over NVA's (Network Virtual Applicances) to get its traffic inspected.
Spoke 2 workloads shoiuld breakout should be foreced tunneling through Express Route and internet breakout to there On-premises firewall. 

## The following diagram illustrates these two traffic flow (forced tunneled and non



In this example, the Frontend subnet is not force tunneled. The workloads in the Frontend subnet can continue to accept and respond to customer requests from the Internet directly. The Mid-tier and Backend subnets are forced tunneled. Any outbound connections from these two subnets to the Internet will be forced or redirected back to an on-premises site via one of the Site-to-site (S2S) VPN tunnels.

This allows you to restrict and inspect Internet access from your virtual machines or cloud services in Azure, while continuing to enable your multi-tier service architecture required. If there are no Internet-facing workloads in your virtual networks, you also can apply forced tunneling to the entire virtual networks.
