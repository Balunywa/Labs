# High availibility is not the same as disaster recovery

## Intro:

This example scenarios are applicable to any industry that needs to deploy resilient Azure Networking built for high availability and disaster recovery. In this scenario, we look at Azure classic Hub & Spoke and Azure Front in a two region logical layout.

When designing your Azure Network to be resilient, you must understand your availability and disaster recovery requirements. How much downtime is acceptable? The amount of downtime is partly a function of cost. How much will potential downtime cost your business? How much should you invest in making the Network highly available and resilient to failuer?

High availability and disaster recovery follow some of the same best practices and the strategies can be applied at all levels of the architecture. Some mitigations are more tactical and others more strategic in nature for example:

- Retrying a remote call after a transient network failure.
- Failing over the entire network to a secondary region.
- Having the right monitoring and diagnostics to to detect failures when they happen, and to find the root causes
 
It's rare for an entire region to experience a disruption, but transient problems such as network congestion are more common so target these issues first, whereas availability focuses on components of the workload, disaster recovery focuses on discrete copies of the entire workload. Hence DR (Disaster Recovery) has different objectives from HA (High Availability), measuring time to recovery after the larger scale events that qualify as disasters. 

You should ensure your workload meets your availability objectives, and commitments you make to your customers. Architecting HA and DR into your Network ensures your workloads are available and can recover from failures at any scale. So your DR design and strategy will require a different apporach than those for HA, focused  on deploying individually separate and distinct network components to multiple locations, so that you can fail over the entire network if necessary to a different region. 

## HA & DR Virtual Network Hub & Spoke Express Route With and Azure Front Door

![image](https://user-images.githubusercontent.com/81341827/183797963-a862c0ee-4533-49d7-aad5-fdb08fbe3461.png)



## HA & DR Virtual Network Hub & Spoke Azure Express Route, S2S VPN and Azure Front Door

![image](https://user-images.githubusercontent.com/81341827/183798668-7cd4d1f6-8eb1-4f57-a778-8e688f669f30.png)


Both availability and disaster recovery rely on some of the same best practices such as:

Monitoring for failures
Deploying to multiple locations
Automatic failover. 

Its important to note that, whereas Availability focuses on components of the workload, disaster recovery focuses on discrete copies of the entire workload. Hence DR (Disaster Recovery) has different objectives from HA (High Availability), measuring time to recovery after the larger scale events that qualify as disasters. You should first ensure your workload meets your availability objectives, as a highly available architecture will enable you to meet customers’ needs in the event of availability impacting events. Your disaster recovery strategy requires different approaches than those for availability, focusing on deploying discrete systems to multiple locations, so that you can fail over the entire workload if necessary.

The goal of this lab is to demostrate how to use an Apache web server behind an Application Gateway to serve files stored on Azure File Shares. It can be a single VM,or multiple VM's serving files from Azure Files.

## Lab Diagram
The lab uses a single virtual nework and two subnets one for the Application Gateway and the second one for the Apache Web Server and Files Share private end point. Additionally it uses Azure supporting services for backing up the files shares, monitoring and overall security.

![image](https://user-images.githubusercontent.com/81341827/180940823-1d03c9a3-bdb5-425f-8cf5-d37a0c6443d3.png)

## Components

- One Virtual Network and two subnets (one for Application Gatewat and the second VM's and Private end points)
- Azure Linux VM in avilibality sets
- Storage Account for Azure File Shares
- Private end point deployed for the Azure Files

## Deploy this solution

The lab is also available in the below .azcli that you can run and execute. You can open Azure Cloud Shell (Bash) and run the following commands to build the lab:

## Create resource group

In Azure, you allocate related resources to a resource group. Create a resource group by using `az group create`. 

The following example creates a resource group named *myResourceGroupAG* in the *eastus* location.

```azurecli-interactive 
az group create --name myResourceGroupAG --location eastus
```

## Create network resources 

For Azure to communicate between the resources that you create, it needs a virtual network.  The application gateway subnet can contain only application gateways. No other resources are allowed.  You can either create a new subnet for Application Gateway or use an existing one. In this example, you create two subnets: one for the application gateway, and another for the backend servers and private end point. You can configure the Frontend IP of the Application Gateway to be Public or Private as per your use case. In this example, you'll choose a Public Frontend IP address.

To create the virtual network and subnet, use `az network vnet create`. Run `az network public-ip create` to create the public IP address.

```azurecli-interactive
az network vnet create \
  --name myVNet \
  --resource-group myResourceGroupAG \
  --location eastus \
  --address-prefix 10.21.0.0/16 \
  --subnet-name myAGSubnet \
  --subnet-prefix 10.21.0.0/24
az network vnet subnet create \
  --name myBackendSubnet \
  --resource-group myResourceGroupAG \
  --vnet-name myVNet   \
  --address-prefix 10.21.1.0/24
az network public-ip create \
  --resource-group myResourceGroupAG \
  --name myAGPublicIPAddress \
  --allocation-method Static \
  --sku Standard
```
## Create Storage Account
To create a storage account using the Azure CLI, we will use the az storage account create command. This command has many options; only the required options are shown. To learn more about the advanced options, see the [`az storage account create` command documentation](/cli/azure/storage/account).

To simplify the creation of the storage account and subsequent file share, we will store several parameters in variables. You may replace the variable contents with whatever values you wish, however note that the storage account name must be globally unique.

```azurecli
resourceGroupName="myResourceGroupAG"
storageAccountName="mystorageacct"
region="eastus"
```

To create a storage account capable of storing premium Azure file shares, we will use the following command. Note that the `--sku` parameter has changed to include both `Premium` and the desired redundancy level of locally redundant (`LRS`). The `--kind` parameter is `FileStorage` instead of `StorageV2` because premium file shares must be created in a FileStorage storage account instead of a GPv2 storage account.

```azurecli
az storage account create \
    --resource-group $resourceGroupName \
    --name $storageAccountName \
    --kind FileStorage \
    --sku Premium_LRS \
    --output none
```
## Create file share
You can create an Azure file share with the [`az storage share-rm create`](/cli/azure/storage/share-rm#az-storage-share-rm-create) command. The following Azure CLI commands assume you have set the variables `$resourceGroupName` and `$storageAccountName` as defined above in the creating a storage account with Azure CLI section.

> [!Important]  
> For premium file shares, the `--quota` parameter refers to the provisioned size of the file share. The provisioned size of the file share is the amount you will be billed for, regardless of usage. Standard file shares are billed based on usage rather than provisioned size.

```azurecli
shareName="myshare"

az storage share-rm create \
    --resource-group $resourceGroupName \
    --storage-account $storageAccountName \
    --name $shareName \
    --access-tier "TransactionOptimized" \
    --quota 1024 \
    --output none
```
---
## Create private endpoint for the file share
To create a private endpoint for your storage account, you first need to get a reference to your storage account and the virtual network subnet to which you want to add the private endpoint. Replace `<storage-account-resource-group-name>`,  `<storage-account-name>`, `<vnet-resource-group-name>`, `<vnet-name>`, and `<vnet-subnet-name>` below:

```bash

storageAccountResourceGroupName="<storage-account-resource-group-name>"
storageAccountName="<storage-account-name>"
virtualNetworkResourceGroupName="<vnet-resource-group-name>"
virtualNetworkName="<vnet-name>"
subnetName="<vnet-subnet-name>"

# Get storage account ID 
storageAccount=$(az storage account show \
        --resource-group $resourceGroupName \
        --name $storageAccountName \
        --query "id" | \
    tr -d '"')

# Get virtual network ID
virtualNetwork=$(az network vnet show \
        --resource-group $resourceGroupName \
        --name $virtualNetworkName \
        --query "id" | \
    tr -d '"')

# Get subnet ID
subnet=$(az network vnet subnet show \
        --resource-group $resourceGroupName \
        --vnet-name $virtualNetworkName \
        --name $subnetName \
        --query "id" | \
    tr -d '"')
```

To create a private endpoint, you must first ensure that the subnet's private endpoint network policy is set to disabled. Then you can create a private endpoint with the `az network private-endpoint create` command.

```azurecli
# Disable private endpoint network policies
az network vnet subnet update \
        --ids $subnet \
        --disable-private-endpoint-network-policies \
        --output none

# Get virtual network location
region=$(az network vnet show \
        --ids $virtualNetwork \
        --query "location" | \
    tr -d '"')

# Create a private endpoint
privateEndpoint=$(az network private-endpoint create \
        --resource-group $resourceGroupName \
        --name "$storageAccountName-PrivateEndpoint" \
        --location $region \
        --subnet $subnet \
        --private-connection-resource-id $storageAccount \
        --group-id "file" \
        --connection-name "$storageAccountName-Connection" \
        --query "id" | \
    tr -d '"')
```
Creating an Azure private DNS zone enables the original name of the storage account, such as `storageaccount.file.core.windows.net` to resolve to the private IP inside of the virtual network. Although optional from the perspective of creating a private endpoint, it is explicitly required for mounting the Azure file share using an AD user principal or accessing via the REST API.  

```bash
# Get the desired storage account suffix (core.windows.net for public cloud).
# This is done like this so this script will seamlessly work for non-public Azure.
storageAccountSuffix=$(az cloud show \
        --query "suffixes.storageEndpoint" | \
    tr -d '"')

# For public cloud, this will generate the following DNS suffix:
# privatelink.file.core.windows.net.
dnsZoneName="privatelink.file.$storageAccountSuffix"

# Find a DNS zone matching desired name attached to this virtual network.
possibleDnsZones=""
possibleDnsZones=$(az network private-dns zone list \
        --query "[?name == '$dnsZoneName'].id" \
        --output tsv)

dnsZone=""
possibleDnsZone=""
for possibleDnsZone in $possibleDnsZones
do
    possibleResourceGroupName=$(az resource show \
            --ids $possibleDnsZone \
            --query "resourceGroup" | \
        tr -d '"')
    
    link=$(az network private-dns link vnet list \
            --resource-group $possibleResourceGroupName \
            --zone-name $dnsZoneName \
            --query "[?virtualNetwork.id == '$virtualNetwork'].id" \
            --output tsv)
    
    if [ -z $link ]
    then
        echo "1" > /dev/null
    else 
        dnsZoneResourceGroup=$possibleResourceGroupName
        dnsZone=$possibleDnsZone
        break
    fi  
done

if [ -z $dnsZone ]
then
    # No matching DNS zone attached to virtual network, so create a new one
    dnsZone=$(az network private-dns zone create \
            --resource-group $virtualNetworkResourceGroupName \
            --name $dnsZoneName \
            --query "id" | \
        tr -d '"')
    
    az network private-dns link vnet create \
            --resource-group $virtualNetworkResourceGroupName \
            --zone-name $dnsZoneName \
            --name "$virtualNetworkName-DnsLink" \
            --virtual-network $virtualNetwork \
            --registration-enabled false \
            --output none
    
    dnsZoneResourceGroup=$resourceGroupName
fi
```

Now that you have a reference to the private DNS zone, you must create an A record for your storage account.

```azurecli
privateEndpointNIC=$(az network private-endpoint show \
        --ids $privateEndpoint \
        --query "networkInterfaces[0].id" | \
    tr -d '"')

privateEndpointIP=$(az network nic show \
        --ids $privateEndpointNIC \
        --query "ipConfigurations[0].privateIpAddress" | \
    tr -d '"')

az network private-dns record-set a create \
        --resource-group $dnsZoneResourceGroup \
        --zone-name $dnsZoneName \
        --name $storageAccountName \
        --output none

az network private-dns record-set a add-record \
        --resource-group $dnsZoneResourceGroup \
        --zone-name $dnsZoneName \
        --record-set-name $storageAccountName \
        --ipv4-address $privateEndpointIP \
        --output none
```

> [!Note]  
> The name of your file share must be all lowercase. For complete details about naming file shares and files, see [Naming and referencing shares, directories, files, and metadata](/rest/api/storageservices/Naming-and-Referencing-Shares--Directories--Files--and-Metadata).

## Create the backend servers

A backend can have NICs, virtual machine scale sets, public IP addresses, internal IP addresses, fully qualified domain names (FQDN), and multi-tenant back-ends like Azure App Service. In this example, you create two virtual machines to use as backend servers for the application gateway. You also mount the Azure File Shares to the backend VM.

#### Create one virtual machines

Create the network interfaces with `az network nic create`. To create the virtual machines, you use `az vm create`.

```azurecli-interactive
for i in `seq 1 2`; do
  az network nic create \
    --resource-group myResourceGroupAG \
    --name myNic$i \
    --vnet-name myVNet \
    --subnet myBackendSubnet
  az vm create \
    --resource-group myResourceGroupAG \
    --name myVM$i \
    --nics myNic$i \
    --image UbuntuLTS \
    --admin-username azureuser \
    --generate-ssh-keys \
    --custom-data cloud-init.txt
done
```
## Connect to the Linux VM you created above

In Azure there are multiple ways to connect to a Linux virtual machine. The most common practice for connecting to a Linux VM is using the Secure Shell Protocol (SSH). This is done via any standard SSH client commonly found in Linux and Windows. You can also use [Azure Cloud Shell](../cloud-shell/overview.md) from any browser.
 
This document describes how to connect, via SSH, to a VM that has a public IP. If you need to connect to a VM without a public IP, see [Azure Bastion Service](../bastion/bastion-overview.md).

## Follow the following setps once logged in
```
sudo apt-get update 
sudo apt-get install nfs-common
sudo apt-get install apache2

```
## The status can be check by running the following commands

```
sudo systemctl status apache2
```
## Create mount point

By default, the document root directory is /var/www/html. You will mount your Azure file shares on a subdirectory under the document root.

Create a subdirectory named smb-azure-files or nfs-azure-files or any name of your choice to use as the mount point for your file system, under /var/www/html.

```
sudo mkdir /var/www/html/nfs-azure-files
```
Mount your Azure file share using the following command. replace storage url with the storage url of your file system.
```
sudo mount -t nfs your-storage-acct-name.file.core.windows.net:/your-storage-acct-name/your-filesharename /var/www/html/nfs-azure-files -o vers=4,minorversion=1,sec=sys
```
Create a sample html file or any file type.

Change directory to the mount point.

```
cd /var/www/html/nfs-azure-files
```
Make a subdirectory called testdir.
```
sudo mkdir testdir  
```
Change directory so you can create files in the testdir subdirectory.
```
cd testdir
```
Create a test hello.html file.
```
echo "<html><h1>Hello from Azure Files Shares</h1></html>" > hello.html
```
## Create the application gateway

Create an application gateway using `az network application-gateway create`. When you create an application gateway with the Azure CLI, you specify configuration information, such as capacity, SKU, and HTTP settings. Azure then adds the private IP addresses of the network interfaces as servers in the backend pool of the application gateway.

```azurecli-interactive
address1=$(az network nic show --name myNic1 --resource-group myResourceGroupAG | grep "\"privateIpAddress\":" | grep -oE '[^ ]+$' | tr -d '",')
address2=$(az network nic show --name myNic2 --resource-group myResourceGroupAG | grep "\"privateIpAddress\":" | grep -oE '[^ ]+$' | tr -d '",')
az network application-gateway create \
  --name myAppGateway \
  --location eastus \
  --resource-group myResourceGroupAG \
  --capacity 2 \
  --sku Standard_v2 \
  --public-ip-address myAGPublicIPAddress \
  --vnet-name myVNet \
  --subnet myAGSubnet \
  --servers "$address1" "$address2" \
  --priority 100


## Clean up resources

When you're done, delete the resource group. Deleting the resource group deletes the storage account, the Azure file share, and any other resources that you deployed inside the resource group.
