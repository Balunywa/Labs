# Forced Tunneling and Internet Breakout with Express Route & NVA Scenario

Forced tunneling lets you redirect or "force" all Internet-bound traffic back to your on-premises location via a Site-to-Site VPN tunnel for inspection and auditing. This is a critical security requirement for most enterprise IT policies. If you don't configure forced tunneling, Internet-bound traffic from your VMs in Azure always traverses from the Azure network infrastructure directly out to the Internet, without the option to allow you to inspect or audit the traffic. Unauthorized Internet access can potentially lead to information disclosure or other types of security breaches.

Forced tunneling can be configured by using Azure PowerShell. It can't be configured using the Azure portal. This article helps you configure forced tunneling for virtual networks created using the [Resource Manager deployment model](../azure-resource-manager/management/deployment-models.md). If you want to configure forced tunneling for the classic deployment model, see [Forced tunneling - classic](vpn-gateway-about-forced-tunneling.md).

## About forced tunneling

The following diagram illustrates how forced tunneling works.