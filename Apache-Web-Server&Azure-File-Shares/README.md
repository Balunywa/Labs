
# Lab: Set up an Apache web server to serve Azure files shares

## Intro: 

The gola of this lab is to demostrate how to use an Apache web server behind an Application Gateway to serve files stored on Azure File Shares. It can be a single VM,or multiple VM's serving files from Azure Files.

## Lab Diagram
The lab uses a single virtual nework and two subnets one for the Application Gateway and the second one for the Apache Web Server and Files Share private end point. Additionally it uses Azure supporting services for backing up the files shares, monitoring and overall security.

![image](https://user-images.githubusercontent.com/81341827/180940823-1d03c9a3-bdb5-425f-8cf5-d37a0c6443d3.png)

### Components

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
storageAccountName="mystorageacct$RANDOM"
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


## Create the backend servers

A backend can have NICs, virtual machine scale sets, public IP addresses, internal IP addresses, fully qualified domain names (FQDN), and multi-tenant back-ends like Azure App Service. In this example, you create one virtual machines to use as backend servers for the application gateway. You also mount the Azure File Shares to the backend VM.

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
```

It can take up to 30 minutes for Azure to create the application gateway. After it's created, you can view the following settings in the **Settings** section of the **Application gateway** page:

- **appGatewayBackendPool**: Located on the **Backend pools** page. It specifies the required backend pool.
- **appGatewayBackendHttpSettings**: Located on the **HTTP settings** page. It specifies that the application gateway uses port 80 and the HTTP protocol for communication.
- **appGatewayHttpListener**: Located on the **Listeners page**. It specifies the default listener associated with **appGatewayBackendPool**.
- **appGatewayFrontendIP**: Located on the **Frontend IP configurations** page. It assigns *myAGPublicIPAddress* to **appGatewayHttpListener**.
- **rule1**: Located on the **Rules** page. It specifies the default routing rule that's associated with **appGatewayHttpListener**.

## Test the application gateway

Although Azure doesn't require an NGINX web server to create the application gateway, you installed it in this quickstart to verify whether Azure successfully created the application gateway. To get the public IP address of the new application gateway, use `az network public-ip show`. 

```azurecli-interactive
az network public-ip show \
  --resource-group myResourceGroupAG \
  --name myAGPublicIPAddress \
  --query [ipAddress] \
  --output tsv
```

Copy and paste the public IP address into the address bar of your browser.
​    
![Test application gateway](./media/quick-create-cli/application-gateway-nginxtest.png)

When you refresh the browser, you should see the name of the second VM. This indicates the application gateway was successfully created and can connect with the backend.

## Access the file shares

![image](https://user-images.githubusercontent.com/81341827/180942210-ed7441ff-9755-4ff9-92ff-08754a403d44.png)


## Clean up resources

When you're done, delete the resource group. Deleting the resource group deletes the storage account, the Azure file share, and any other resources that you deployed inside the resource group.


## Next steps

> [!div class="nextstepaction"]
> [Learn about using NFS Azure file shares](files-nfs-protocol.md)
