# Overview

The API Management service supports both Virtual Network support and Non Virtual Network support depending on the SKU type that is selected https://azure.microsoft.com/en-us/pricing/details/api-management/,Virtual Network in internal mode, which makes it accessible only from within the Virtual Network and Non Virtual Network mode which makes it accessible only externally over a public end point. Azure Application Gateway is a PAAS Service, which provides a Layer-7 load balancer. It acts as a reverse-proxy service and provides among its offering a Web Application Firewall (WAF).

Combining API Management provisioned in an Non Virtual Network with the Application Gateway frontend enables the following scenarios:

Use a single API Management resource and APIs defined in API Management available for external consumers.


## Scenario

This article covers how to use a non Virtual Networ single API Management service for external consumers and make it act as a single frontend. You will also see how to expose your APIs for External Consumption using routing functionality available in Application Gateway.

In this setup example all your APIs are managed without a Virtual Network. All traffic comes in externally from the internet.

