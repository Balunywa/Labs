# Integrate Standard APIM SKU without VNET support with Application Gateway

Cost and VNET integration support are some of the main considerations when selecting the APIM SKU for production use cases. The API Management service supports both Virtual Networks and Non Virtual Networks deployments depending on the SKU type that is selected. Of-course it will cost you more if you went with the premium SKU mainly for the need to support VNET integration. However, you can still go with the standard SKU which does not support VNET integration and protect your APIM’s behind an AZ-App-GW/WAF which in turn reduces your overall cost and securely protects your API’s.

APIM deployed with Virtual Network in internal mode makes it accessible only from within the Virtual Network, and with Non Virtual Network mode it makes it accessible only externally over a public end point, and both of this scenarios are deployed and configured differently if you are using and App-GW/WAF.
Azure Application Gateway is a PAAS Service, which provides a Layer-7 load balancer. It acts as a reverse-proxy service and provides among its offering a Web Application Firewall (WAF).

Combining API Management provisioned in an Non Virtual Network with the Application Gateway frontend enables the following scenarios:
Use a single API Management resource and APIs defined in API Management available for external consumers.


## Scenario

This article covers how to use a non Virtual Network single API Management service for external consumers and make it act as a single frontend. You will also see how to expose your APIs for External Consumption using routing functionality available in Application Gateway. 

In this setup example all your APIs are managed without a Virtual Network. All traffic comes in externally from the internet.

![image](https://user-images.githubusercontent.com/81341827/121389809-c49eb480-c91a-11eb-95ea-04a09fec2803.png)

## Prerequisites

- Azure Key Vault - To store the custom certificates that will be used on AZ-APP-GW and AZ-APIM
- Managed identity - For App-GW and APIM to access the KeyVault to retirve the certificates
- Custom Domains - In my case am using GoDaddy as my domin provider, but you can also buy App Sevrice domains, and use the Azure Domain Name    System to manage your domains, all within Azure. Azure DNS also gives you a range of secure, reliable domain hosting options. https://docs.microsoft.com/en-us/azure/app-service/manage-custom-dns-buy-domain
- TLS certificates - You can use selfsigned certificates, import your CA singned certificates, in this example a private App Service certificate that’s managed by Azure. You can export copies, and use them with other Azure Services like AZ-APP-GW and AZ-APIM https://docs.microsoft.com/en-us/azure/app-service/configure-ssl-certificate#start-certificate-order
- Make sure that you are using the latest version of Azure PowerShell. See the installation instructions at Install Azure PowerShell.

## Use AZ-Cloud Shell for Certificate Generation and Export to KeyVault when using App Service Certificate
# Fetch the secret from Azure Key Vault
$secret = Get-AzKeyVaultSecret -VaultName 'certs-kv-one' -Name 'onemscont998bf559-e98d-45c5-93b1-41e5b827ea82/236a6618f3fc4eb4b592fbe02a0ebf7e'

# Convert the secret value to plain text from SecureString
$secretValue = $secret.SecretValue | ConvertFrom-SecureString -AsPlainText

# Create an X509Certificate2 object from the secret
$pfxCertObject = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2 -ArgumentList @(
    [Convert]::FromBase64String($secretValue), 
    "", 
    [System.Security.Cryptography.X509Certificates.X509KeyStorageFlags]::Exportable
)

# Generate a random PFX password
$pfxPassword = -join ((65..90) + (97..122) + (48..57) | Get-Random -Count 50 | ForEach-Object { [char]$_ })

# Define the current directory
$currentDirectory = (Get-Location -PSProvider FileSystem).ProviderPath

# Set the environment's current directory (this seems redundant and could potentially be removed)
[Environment]::CurrentDirectory = $currentDirectory

# Export the certificate as PFX and save to the current directory
$pfxFilePath = Join-Path -Path $currentDirectory -ChildPath 'appservicecertificate.pfx'
[io.file]::WriteAllBytes($pfxFilePath, $pfxCertObject.Export([System.Security.Cryptography.X509Certificates.X509ContentType]::Pkcs12, $pfxPassword))

# Output details to the user
Write-Host "Created an App Service Certificate copy at: $pfxFilePath"
Write-Warning "For security reasons, do not store the PFX password. Use it directly from the console as required."
Write-Host "PFX password: $pfxPassword"


## Save the generated password to use when uploading the certificate to KeyVault, downlaod the certificate from AZ-Cloud-Shell
![image](https://user-images.githubusercontent.com/81341827/121402584-113cbc80-c928-11eb-99ef-c803ee5d8a86.png)

## API Management setup
This quick start describes the steps for creating a new API Management instance using the Azure portal. In this example we are using the APIM standard SKU, and the developer portal and API gateway will be accessible from the Internet. Make sure your select the standard SKU for the pricing tier. https://docs.microsoft.com/en-us/azure/api-management/get-started-create-service-instance

The created API Management service instance by default is available through *.azure-api.net subdomain (for example, contoso.azure-api.net). You will have to expose the service through your own domain name, such as contoso.com.

## Deafult Domain Mappings:
  - <apim-service-name>.azure-api.net
  - <apim-service-name>.developer.azure-api.net
  
## Custom Domain Mappings/With TLS Certificates fetched from AZ-KeyVault
  - api.contoso.com
  - portal.contoso.com
  
![image](https://user-images.githubusercontent.com/81341827/121414342-86ae8a00-c934-11eb-8dfa-c7208170012b.png)


## Application Gateway setup
In this quick start, you use the Azure portal to create an application gateway. Make sure you select add backed end without target https://docs.microsoft.com/en-us/azure/application-gateway/quick-create-portal
  
## CNAME Mappings
  
Once the gateway is created, the next step is to configure the front end for communication. When using a public IP, Application Gateway requires a dynamically assigned DNS name which you can create through the portal, The Application Gateway’s DNS name should be used to create a CNAME record which points the APIM proxy host name (e.g.api.contoso.com). The use of A-records is not a good idea, since the VIP may change on restart of gateway

## Domain Provider Mappings:

- api.contoso.com - CNAME mapped to APP-GW FQDN (This Will redirect all external incoming traffic for the APIM-Proxy to APP-GW for inspection)
- portal.contoso.com - CNAME mapped to APP-GW FQDN (This will redirect all external incoming traffic for the APIM-Developer portal to APP-GW for inspection)
- backportal.contoso.com - CNAME mapped to the default Dev-APIM FQDN (This will be used as the Devportal backend in APP-GW)
 
## APP-GW Backend Mappings
  
- api.azure-api.net - Deafult APIM-GW Proxy FQDN still resolves traffic.
- backportal.contoso.com - CNAME mapped Dev-Portal APIM FQDN
  
![image](https://user-images.githubusercontent.com/81341827/121415529-d6da1c00-c935-11eb-81d7-91ae08cd062d.png)

  
## APP-GW Listerner Mappings
  
- api.contoso.com - CNAME mapped to APP-GW FQDN.
- portal.contoso.com - CNAME mapped to APP-GW FQDN.
  
![image](https://user-images.githubusercontent.com/81341827/121418288-cbd4bb00-c938-11eb-97c0-8be1ce8c6169.png)

  
## The APP-GW health Probes path for APIM-Porxy will be /status-0123456789abcdef and for the Devloper portal will be a forward /
  
![image](https://user-images.githubusercontent.com/81341827/121419174-adbb8a80-c939-11eb-9cc5-04c913f9d9d6.png)

## API Management Inboud Policy to only allow traffic coming in from the APP-GW Public IP address
  
The ip-filter policy filters (allows/denies) calls from specific IP addresses and/or address ranges. In the following example, the policy only allows requests coming from the application gateway public ip address. https://docs.microsoft.com/en-us/azure/api-management/api-management-access-restriction-policies#RestrictCallerIPs
  
![image](https://user-images.githubusercontent.com/81341827/121420195-ca0bf700-c93a-11eb-9ea0-3372255e9ece.png)


### Policy statement

```xml
<ip-filter action="allow | forbid">
    <address>Your-APP-GW-Public-IP</address>
</ip-filter>
```
  
  



