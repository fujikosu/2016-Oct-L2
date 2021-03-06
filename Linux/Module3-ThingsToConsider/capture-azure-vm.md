# How to capture a Linux virtual machine to use as a Resource Manager template

Use the Azure Command-Line Interface (CLI) to capture and generalize an Azure virtual machine running Linux. You can then use it as an Azure Resource Manager template to create other virtual machines. This template specifies the OS disk and data disks attached to the virtual machine. It doesn't include the virtual network resources you need to create an Azure Resource Manager VM. So, you'll usually need to set those up separately before you create another virtual machine that uses the template.

## Capture the VM

1. When you are ready to capture the VM, connect to it using your SSH client.

2. In the SSH window, type the following command. The output from **waagent** may vary slightly depending on the version of this utility:

	`sudo waagent -deprovision+user`

	This command attempts to clean the system and make it suitable for reprovisioning. This operation performs the following tasks:

	- Removes SSH host keys (if Provisioning.RegenerateSshHostKeyPair is 'y' in the configuration file)
	- Clears nameserver configuration in /etc/resolvconf
	- Removes the `root` user's password from /etc/shadow (if Provisioning.DeleteRootPassword is 'y' in the configuration file)
	- Removes cached DHCP client leases
	- Resets host name to localhost.localdomain
	- Deletes the last provisioned user account (obtained from /var/lib/waagent) and associated data.


NOTE: Deprovisioning deletes files and data to generalize the image. Only run this command on a VM that you intend to capture as an image. It does not guarantee that the image is cleared of all sensitive information or is suitable for redistribution to third parties.

3. Type **y** to continue. You can add the **-force** parameter to avoid this confirmation step.

4. Type **exit** to close the SSH client.

5. From your client computer, open the Azure CLI and login to your Azure subscription. For details, read [Connect to an Azure subscription from the Azure CLI](https://azure.microsoft.com/en-us/documentation/articles/xplat-cli-connect/).

6. Make sure you are in Resource Manager mode:

	`azure config mode arm`

7. Stop the VM that you already deprovisioned by using the following command:

	`azure vm deallocate -g <your-resource-group-name> -n <your-virtual-machine-name>`

8. Generalize the VM with the following command:

	`azure vm generalize -g <your-resource-group-name> -n <your-virtual-machine-name>`

9. Now capture the image and a local file template with the following command:

	`azure vm capture <your-resource-group-name>  <your-virtual-machine-name> <your-vhd-name-prefix> -t <path-to-your-template-file-name.json>`

	This command creates a generalized OS image, using the VHD name prefix you specify for the VM disks. The image VHD files get created by default in the same storage account that the original VM used. (The VHDs for any new VMs you create from the image will be stored in the same account.) The **-t** option creates a local JSON file template you can use to create a new VM from the image.

TIP: To find the location of an image, open the JSON file template. In the **storageProfile**, find the **uri** of the **image** located in the **system** container. For example, the uri of the OS disk image is similar to `https://xxxxxxxxxxxxxx.blob.core.windows.net/system/Microsoft.Compute/Images/vhds/<your-vhd-name-prefix>-osDisk.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.vhd`.

## Deploy a new VM from the captured image
Now use the image with a template to create a Linux VM. These steps show you how to use the Azure CLI and the JSON file template you created with the `azure vm capture` command to create the VM in a new virtual network.

### Create network resources

To use the template, you first need to set up a virtual network and NIC for your new VM. We recommend you create a new resource group for these resources in the location where your VM image is stored. Run commands similar to the following, substituting names for your resources and an appropriate Azure location ("centralus" in these commands):

	azure group create <your-new-resource-group-name> -l "centralus"

	azure network vnet create <your-new-resource-group-name> <your-vnet-name> -l "centralus"

	azure network vnet subnet create <your-new-resource-group-name> <your-vnet-name> <your-subnet-name>

	azure network public-ip create <your-new-resource-group-name> <your-ip-name> -l "centralus"

	azure network nic create <your-new-resource-group-name> <your-nic-name> -k <your-subnetname> -m <your-vnet-name> -p <your-ip-name> -l "centralus"

To deploy a VM from the image by using the JSON you saved during capture, you'll need the Id of the NIC. Obtain it by running the following command.

	azure network nic show <your-new-resource-group-name> <your-nic-name>

The **Id** in the output is a string similar to the following.

	/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/<your-new-resource-group-name>/providers/Microsoft.Network/networkInterfaces/<your-nic-name>



### Create a new deployment
Now run the following command to create your VM from the captured VM image and the template JSON file you saved.

	azure group deployment create <your-new-resource-group-name> <your-new-deployment-name> -f <path-to-your-template-file-name.json>

You are prompted to supply a new VM name, the admin user name and password, and the Id of the NIC you created previously.

	info:    Executing command group deployment create
	info:    Supply values for the following parameters
	vmName: mynewvm
	adminUserName: myadminuser
	adminPassword: ********
	networkInterfaceId: /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resource Groups/mynewrg/providers/Microsoft.Network/networkInterfaces/mynewnic

You will see output similar to the following for a successful deployment.

	+ Initializing template configurations and parameters
	+ Creating a deployment
	info:    Created template deployment "dlnewdeploy"
	+ Waiting for deployment to complete
	data:    DeploymentName     : mynewdeploy
	data:    ResourceGroupName  : mynewrg
	data:    ProvisioningState  : Succeeded
	data:    Timestamp          : 2015-10-29T16:35:47.3419991Z
	data:    Mode               : Incremental
	data:    Name                Type          Value


	data:    ------------------  ------------  -------------------------------------

	data:    vmName              String        mynewvm


	data:    vmSize              String        Standard_D1


	data:    adminUserName       String        myadminuser


	data:    adminPassword       SecureString  undefined


	data:    networkInterfaceId  String        /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/mynewrg/providers/Microsoft.Network/networkInterfaces/mynewnic
	info:    group deployment create command OK

### Verify the deployment

Now SSH to the virtual machine you created to verify the deployment and start using the new VM. To connect via SSH, find the IP address of the VM you created by running the following command:

	azure network public-ip show <your-new-resource-group-name> <your-ip-name>

The public IP address is listed in the command output. By default you connect to the Linux VM by SSH on port 22.

## Create additional VMs with the template

Use the captured image and template to deploy additional VMs using the steps outlined in the preceding section.

* Ensure that your VM image is in the same storage account that will host your VM's VHD
* Copy the template JSON file and enter a unique value for the **uri** of each VM's VHD
* Create a NIC in either the same or a different virtual network
* Create a deployment in the resource group in which you set up the virtual network, using the modified template JSON file

## Use the azure vm create command

You'll generally want to use a Resource Manager template to create a VM from the image. However, you can create the VM _imperatively_ by using the **azure vm create** command with the **-Q** (**--image-urn**) parameter. If you use this method, you also pass the **-d** (**--os-disk-vhd**) parameter to specify the location of the OS .vhd file for the new VM. This must be in the vhds container of the storage account where the image VHD file is stored. The command copies the VHD for the new VM automatically to the vhds container.

Do the following before running **azure vm create** with the image:

1.	Create a resource group, or identify an existing resource group for the deployment.

2.	Create a public IP address resource and a NIC resource for the new VM. For steps to create a virtual network, public IP address, and NIC by using the CLI, see earlier in this article. (**azure vm create** can also create a NIC but you will need to pass additional parameters for a virtual network and subnet.)


Then run a command similar to the following, passing URIs to both the new OS VHD file and the existing image.

	azure vm create <your-resource-group-name> <your-new-vm-name> eastus Linux -d "https://xxxxxxxxxxxxxx.blob.core.windows.net/vhds/<your-new-VM-prefix>.vhd" -Q "https://xxxxxxxxxxxxxx.blob.core.windows.net/system/Microsoft.Compute/Images/vhds/<your-image-prefix>-osDisk.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.vhd" -z Standard_A1 -u <your-admin-name> -p <your-admin-password> -f <your-nic-name>

For additional command options, run `azure help vm create`.
