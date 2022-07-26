
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

### Deploy this solution

The lab is also available in the above .azcli that you can rename as .sh (shell script) and execute. You can open Azure Cloud Shell (Bash) and run the following commands build the entire lab:

```bash
wget -O irazfw-deploy.sh https://raw.githubusercontent.com/dmauser/azure-virtualwan/main/inter-region-azfw/irazfw-deploy.azcli
chmod +xr irazfw-deploy.sh
./irazfw-deploy.sh 
```

## Access the file shares

![image](https://user-images.githubusercontent.com/81341827/180942210-ed7441ff-9755-4ff9-92ff-08754a403d44.png)


## Clean up resources

When you're done, delete the resource group. Deleting the resource group deletes the storage account, the Azure file share, and any other resources that you deployed inside the resource group.


## Next steps

> [!div class="nextstepaction"]
> [Learn about using NFS Azure file shares](files-nfs-protocol.md)