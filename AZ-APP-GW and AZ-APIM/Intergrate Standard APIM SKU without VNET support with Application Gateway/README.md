# Overview

The API Management service supports both Virtual Network support and Non Virtual Network depending on the SKU type that is selected https://azure.microsoft.com/en-us/pricing/details/api-management/, With Virtual Network in internal mode it makes it accessible only from within the Virtual Network and with Non Virtual Network mode it makes it accessible only externally over a public end point. Azure Application Gateway is a PAAS Service, which provides a Layer-7 load balancer. It acts as a reverse-proxy service and provides among its offering a Web Application Firewall (WAF).

Combining API Management provisioned in an Non Virtual Network with the Application Gateway frontend enables the following scenarios:

Use a single API Management resource and APIs defined in API Management available for external consumers.


## Scenario

This article covers how to use a non Virtual Networ single API Management service for external consumers and make it act as a single frontend. You will also see how to expose your APIs for External Consumption using routing functionality available in Application Gateway. 

In this setup example all your APIs are managed without a Virtual Network. All traffic comes in externally from the internet.

![image](https://user-images.githubusercontent.com/81341827/121389809-c49eb480-c91a-11eb-95ea-04a09fec2803.png)

## Prerequisites

- Azure Key Vault - To store the custom certificates that will be used on AZ-APP-GW and AZ-APIM
- Managed identity - For App-GW and APIM to access the KeyVault to retirve the certificates
- Custom Domains - In my case am using GoDaddy as my domin provider, but you can also buy App Sevrice domains, and use the Azure Domain Name    System to manage your domains, all within Azure. Azure DNS also gives you a range of secure, reliable domain hosting options. https://docs.microsoft.com/en-us/azure/app-service/manage-custom-dns-buy-domain
- TLS certificates - You can use selfsigned certificates, import your CA singned certificates, in this example a private App Service certificate that’s managed by Azure. You can export copies, and use them with other Azure Services like AZ-APP-GW and AZ-APIM https://docs.microsoft.com/en-us/azure/app-service/configure-ssl-certificate#start-certificate-order
- Make sure that you are using the latest version of Azure PowerShell. See the installation instructions at Install Azure PowerShell.

## Use AZ-Cloud Shell for Certificate Generation and Export to KeyVault when using App Service Certificate

$secret = Get-AzKeyVaultSecret -VaultName YourKeyVaultName  -Name YourCertificateSecretName
$secretValue = $secret.SecretValue | ConvertFrom-SecureString -AsPlainText
$pfxCertObject= New-Object System.Security.Cryptography.X509Certificates.X509Certificate2 -ArgumentList @([Convert]::FromBase64String($secretValue),"",[System.Security.Cryptography.X509Certificates.X509KeyStorageFlags]::Exportable)
$pfxPassword = -join ((65..90) + (97..122) + (48..57) | Get-Random -Count 50 | % {[char]$_})
$currentDirectory = (Get-Location -PSProvider FileSystem).ProviderPath
[Environment]::CurrentDirectory = (Get-Location -PSProvider FileSystem).ProviderPath
[io.file]::WriteAllBytes(".\appservicecertificate.pfx",$pfxCertObject.Export([System.Security.Cryptography.X509Certificates.X509ContentType]::Pkcs12,$pfxPassword))

Write-Host "Created an App Service Certificate copy at: $currentDirectory\appservicecertificate.pfx"
Write-Warning "For security reasons, do not store the PFX password. Use it directly from the console as required."
Write-Host "PFX password: $pfxPassword"

## Save the generated password to use when uploading the certificate to KeyVault, downlaod the certificate from AZ-Cloud-Shell
![image](https://user-images.githubusercontent.com/81341827/121402584-113cbc80-c928-11eb-99ef-c803ee5d8a86.png)

## API Management setup
This quickstart describes the steps for creating a new API Management instance using the Azure portal. In tis example we are using the APIM standard SKU and the The developer portal and API gateway will be accessible from the Internet. Make sure your select the standard SKU for the pricing tier. https://docs.microsoft.com/en-us/azure/api-management/get-started-create-service-instance

The created API Management service instance by default is available through *.azure-api.net subdomain (for example, contoso.azure-api.net). You will have to expose the service through your own domain name, such as contoso.com. MY example am using taacs.cloud as my domain

Deafult Domain Mappings:
  - <apim-service-name>.azure-api.net
  - <apim-service-name>.developer.azure-api.net
  
Custom Domain Mappings/With TLS Certificates fetched from AZ-KeyVault
  - api.contoso.com
  - portal.contoso.com
  
![image](https://user-images.githubusercontent.com/81341827/121414342-86ae8a00-c934-11eb-8dfa-c7208170012b.png)


## Application Gatway setup
In this quickstart, you use the Azure portal to create an application gateway. Make sure your seletc add backed end pulls without target https://docs.microsoft.com/en-us/azure/application-gateway/quick-create-portal
  
## CNAME Mappings
  
Once the gateway is created, the next step is to configure the front end for communication. When using a public IP, Application Gateway requires a dynamically assigned DNS name wich you can create through the protal, The Application Gateway's DNS name should be used to create a CNAME record which points the APIM proxy host name (e.g. uniapi.taacs.cloud). The use of A-records is not since the VIP may change on restart of gateway.

Domain Mappings Examples:

- api.contoso.com - CNAME mapped to APP-GW FQDN (This Will redirect all external incoming traffic for the APIM-Proxy to APP-GW for inspection)
- portal.contoso.com - CNAME mapped to APP-GW FQDN (This will redirect all external incoming traffic for the APIM-Developer portal to APP-GW for inspection)
- backportal.contoso.com - CNAME mapped to the default Dev-APIM FQDN (This will be used as the Devportal backend in APP-GW)
 
APP-GW Backend Mappings
- api.azure-api.net - Deafult APIM GW Proxy FQDN still resolves traffic. 
![image](https://user-images.githubusercontent.com/81341827/121414786-0b010d00-c935-11eb-862c-f49233f99af7.png)

- backportal.contoso.com - CNAME mapped Dev-Portal APIM FQDN
  
  
APP-GW Listerner Mappings
- api.contoso.com - CNAME mapped to APP-GW FQDN.
- portal.contoso.com - CNAME mapped to APP-GW FQDN.

  



  

  
  



