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
In this example we are using the APIM standard SKU and the The developer portal and API gateway will be accessible from the Internet

By default, your API Management service instance is available through *.azure-api.net subdomain (for example, contoso.azure-api.net). You will have to expose the service through your own domain name, such as contos.com

