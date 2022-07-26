
# Lab: Setting up an Apache web server to serve Azure files shares

## Intro: 

The gola of this lab is to demostrate how to use an Apache web server behind and Application Gateway to serve files stored on Azure File Shares. It can be a single VM,or have multiple VM's serving files from Azure Files.

## Lab Diagram
The lab uses a single virtual nework and two subnets 1 for the Application Gateway and the second one for the Apache Web Server.

![image](https://user-images.githubusercontent.com/81341827/180932754-c6ef9b24-3860-46bc-8da2-75675bea45a4.png)


## Getting started

If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.

Sign in to the [Azure portal](https://portal.azure.com).

### Create a FileStorage storage account

Before you can work with an NFS 4.1 Azure file share, you have to create an Azure storage account with the premium performance tier. Currently, NFS 4.1 shares are only available as premium file shares.

1. On the Azure portal menu, select **All services**. In the list of resources, type **Storage Accounts**. As you begin typing, the list filters based on your input. Select **Storage Accounts**.
1. On the **Storage Accounts** window that appears, choose **+ Create**.
1. On the **Basics** tab, select the subscription in which to create the storage account.
1. Under the **Resource group** field, select **Create new** to create a new resource group to use for this tutorial.
1. Enter a name for your storage account. The name you choose must be unique across Azure. The name also must be between 3 and 24 characters in length, and may include only numbers and lowercase letters.
1. Select a region for your storage account, or use the default region. Azure supports NFS file shares in all the same regions that support premium file storage.
1. Select the *Premium* performance tier to store your data on solid-state drives (SSD). Under **Premium account type**, select *File shares*.
1. Leave replication set to its default value of *Locally-redundant storage (LRS)*.
1. Select **Review + Create** to review your storage account settings and create the account.
1. When you see the **Validation passed** notification appear, select **Create**. You should see a notification that deployment is in progress.

## Deploy an Azure VM running Linux

Next, create an Azure VM running Linux to represent the on-premises server. When you create the VM, a virtual network will be created for you. The NFS protocol can only be used from a machine inside of a virtual network.

1. Select **Home**, and then select **Virtual machines** under **Azure services**.

1. Select **+ Create** and then **+ Virtual machine**.

1. In the **Basics** tab, under **Project details**, make sure the correct subscription and resource group are selected. Under **Instance details**, type *myVM* for the **Virtual machine name**, and select the same region as your storage account. Choose the default Ubuntu Server version for your **Image**. Leave the other defaults. The default size and pricing is only shown as an example. Size availability and pricing is dependent on your region and subscription.


1. Under **Administrator account**, select **SSH public key**. Leave the rest of the defaults.


1. Under **Inbound port rules > Public inbound ports**, choose **Allow selected ports** and then select **SSH (22) and HTTP (80)** from the drop-down.


   > [!IMPORTANT]
   > Setting SSH port(s) open to the internet is only recommended for testing. If you want to change this setting later, go back to the **Basics** tab.  

1. Select the **Review + create** button at the bottom of the page.

1. On the **Create a virtual machine** page, you can see the details about the VM you are about to create. Note the name of the virtual network. When you are ready, select **Create**.

1. When the **Generate new key pair** window opens, select **Download private key and create resource**. Your key file will be download as **myKey.pem**. Make sure you know where the .pem file was downloaded, because you'll need the path to it to connect to your VM.

You'll see a message that deployment is in progress. Wait a few minutes for deployment to complete.

## Create an NFS Azure file share

Now you're ready to create an NFS file share and provide network-level security for your NFS traffic.

### Add a file share to your storage account

1. Select **Home** and then **Storage accounts**.

1. Select the storage account you created.

1. Select **Data storage > File shares** from the storage account pane.

1. Select **+ File Share**.

1. Name the new file share *qsfileshare* and enter "100" for the minimum **Provisioned capacity**, or provision more capacity (up to 102,400 GiB) to get more performance. Select **NFS** protocol, leave **No Root Squash** selected, and select **Create**.


### Set up a private endpoint

Next, you'll need to set up a private endpoint for your storage account. This gives your storage account a private IP address from within the address space of your virtual network.

1. Select the file share *qsfileshare*. You should see a dialog that says *Connect to this NFS share from Linux*. Under **Network configuration**, select **Review options**


1. Next, select **Setup a private endpoint**.

1. Select **+ Private endpoint**.



1. Leave **Subscription** and **Resource group** the same. Under **Instance**, provide a name and select a region for the new private endpoint. Your private endpoint must be in the same region as your virtual network, so use the same region as you specified when creating the V M. When all the fields are complete, select **Next: Resource**.


1. Confirm that the **Subscription**, **Resource type** and **Resource** are correct, and select **File** from the **Target sub-resource** drop-down. Then select **Next: Virtual Network**.

1. Under **Networking**, select the virtual network associated with your VM and leave the default subnet. Select **Yes** for **Integrate with private DNS zone**. Select the correct subscription and resource group, and then select **Next: Tags**.

1. You can optionally apply tags to categorize your resources, such as applying the name **Environment** and the value **Test** to all testing resources. Enter name/value pairs if desired, and then select **Next: Review + create**.

 
1. Azure will attempt to validate the private endpoint. When validation is complete, select **Create**. You'll see a notification that deployment is in progress. After a few minutes, you should see a notification that deployment is complete.

### Disable secure transfer

Because the NFS protocol doesn't support encryption and relies instead on network-level security, you'll need to disable secure transfer.

1. Select **Home** and then **Storage accounts**.

1. Select the storage account you created.

1. Select **File shares** from the storage account pane.

1. Select the NFS file share that you created. Under **Secure transfer setting**, select **Change setting**.

 
1. Change the **Secure transfer required** setting to **Disabled**, and select **Save**. The setting change may take up to 30 seconds to take effect.

## Connect to your VM

Create an SSH connection with the VM.

1. Select **Home** and then **Virtual machines**.

1. Select the Linux VM you created for this tutorial and ensure that its status is **Running**. Take note of the VM's public IP address and copy it to your clipboard.

1. If you are on a Mac or Linux machine, open a Bash prompt. If you are on a Windows machine, open a PowerShell prompt.

1. At your prompt, open an SSH connection to your VM. Replace the IP address with the one from your VM, and replace the path to the `.pem` with the path to where the key file was downloaded.

```console
ssh -i .\Downloads\myVM_key.pem azureuser@20.25.14.85
```

If you encounter a warning that the authenticity of the host can't be established, type **yes** to continue connecting to the VM. Leave the ssh connection open for the next step.

> [!TIP]
> The SSH key you created can be used the next time your create a VM in Azure. Just select the **Use a key stored in Azure** for **SSH public key source** the next time you create a VM. You already have the private key on your computer, so you won't need to download anything.

## Mount the NFS share

Now that you've created an NFS share, to use it you have to mount it on your Linux client.

1. Select **Home** and then **Storage accounts**. 

1. Select the storage account you created.

1. Select **File shares** from the storage account pane and select the NFS file share you created.

1. You should see **Connect to this NFS share from Linux** along with sample commands to use NFS on your Linux distribution and a provided mounting script.

   > [!IMPORTANT]
   > The provided mounting script will mount the NFS share only until the Linux machine is rebooted. To automatically mount the share every time the machine reboots, [add an entry in /etc/fstab](storage-how-to-use-files-linux.md#static-mount-with-etcfstab). For more information, enter the command `man fstab` from the Linux command line.


1. Select your Linux distribution (Ubuntu).

1. Using the ssh connection you created to your VM, enter the sample commands to use NFS and mount the file share.

You have now mounted your NFS share, and it's ready to store files.

## Clean up resources

When you're done, delete the resource group. Deleting the resource group deletes the storage account, the Azure file share, and any other resources that you deployed inside the resource group.

1. Select **Home** and then **Resource groups**.
1. Select the resource group you created for this tutorial.
1. Select **Delete resource group**. A window opens and displays a warning about the resources that will be deleted with the resource group.
1. Enter the name of the resource group, and then select **Delete**.

## Next steps

> [!div class="nextstepaction"]
> [Learn about using NFS Azure file shares](files-nfs-protocol.md)
