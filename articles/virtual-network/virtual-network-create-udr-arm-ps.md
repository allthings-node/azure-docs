---
title: Create a user-defined route to route network traffic through a network virtual appliance - PowerShell | Microsoft Docs
description: Learn how to create a user-defined route to override Azure's default routing by routing network traffic through a network virtual appliance.
services: virtual-network
documentationcenter: na
author: jimdial
manager: jeconnoc
editor: ''
tags: azure-resource-manager


ms.service: virtual-network
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: infrastructure-services
ms.date: 10/16/2017
ms.author: jdial

---

# Create a user-defined route - PowerShell

In this tutorial, learn how to create user-defined routes to route traffic between two [virtual network](virtual-networks-overview.md) subnets through a network virtual appliance. A network virtual appliance is a virtual machine that runs a network application, such as a firewall. To learn more about pre-configured network virtual appliances that you can deploy in an Azure virtual network, see the [Azure Marketplace](https://azuremarketplace.microsoft.com/marketplace/apps/category/networking?page=1&subcategories=appliances).

When you create subnets in a virtual network, Azure creates default [system routes](virtual-networks-udr-overview.md#system-routes) that enable resources in all subnets to communicate with each other, as shown in the following picture:

![Default routes](./media/create-user-defined-route/default-routes.png)

In this tutorial, you create a virtual network with public, private, and DMZ subnets, as shown in the picture that follows. Typically web servers might be deployed to a public subnet, and an application or database server might be deployed to a private subnet. You create a virtual machine to act as a network virtual appliance in the DMZ subnet, and optionally, create a virtual machine in each subnet that communicate through the network virtual appliance. All traffic between the public and private subnets is routed through the appliance, as shown in the following picture:

![User-defined routes](./media/create-user-defined-route/user-defined-routes.png)

This article provides steps to create a user-defined route through the Resource Manager deployment model, which is the deployment model we recommend using when creating user-defined routes. If you need to create a user-defined route (classic), see [Create a user-defined route (classic)](virtual-network-create-udr-classic-ps.md). If you're not familiar with Azure's deployment models, see [Understand Azure deployment models](../azure-resource-manager/resource-manager-deployment-model.md?toc=%2fazure%2fvirtual-network%2ftoc.json). To learn more about user-defined routes, see [User-defined routes overview](virtual-networks-udr-overview.md#user-defined).

## Create routes and network virtual appliance

You can either install and configure the latest version of the PowerShell [AzureRM](https://www.powershellgallery.com/packages/AzureRM/) module on your PC, or just click the **Try it** button in any of the scripts to execute the scripts in the Azure Cloud Shell. The Cloud Shell has the PowerShell AzureRM module installed.
 
1. **Prerequisite**: Create a virtual network with two subnets by completing the steps in [Create a virtual network](#create-a-virtual-network).
2. If running PowerShell from your computer, log in to Azure with your [Azure account](../azure-glossary-cloud-terminology.md?toc=%2fazure%2fvirtual-network%2ftoc.json#account) using the `login-azurermaccount` command. If using the Cloud Shell, you're automatically logged in. The Cloud Shell may need to restart to change over from the Bash shell used when creating the pre-requisite virtual network.
3. Set a few variables used throughout the steps:

    ```azurepowershell-interactive
    $rgName="myResourceGroup"
    $location="eastus"
    ```

4. Create a DMZ subnet in the exiting virtual network:

    ```azurepowershell-interactive
    $virtualNetwork = Get-AzureRmVirtualNetwork `
      -Name myVnet `
      -ResourceGroupName $rgName 
    Add-AzureRmVirtualNetworkSubnetConfig `
      -Name 'DMZ' `
      -AddressPrefix "10.0.2.0/24" `
      -VirtualNetwork $virtualNetwork
    $virtualNetwork | Set-AzureRmVirtualNetwork
    ```

5. Create a static public IP address for the network virtual appliance virtual machine.

    ```azurepowershell-interactive
    $Pip = New-AzureRmPublicIpAddress `
      -AllocationMethod Static `
      -ResourceGroupName $rgName `
      -Location $location `
      -Name myPublicIp-myVm-Nva
    
6. Create a network interface in the *DMZ* subnet, assign a static private IP address to it, and enable IP forwarding for the network interface. By assigning a static IP address to the network interface, you ensure that it doesn't change for the life of the virtual machine the network interface is attached to. IP forwarding must be enabled for each network interface that receives traffic not addressed to an IP address assigned to it.

    ```azurepowershell-interactive
    $virtualNetwork = Get-AzureRmVirtualNetwork `
      -Name myVnet `
      -ResourceGroupName $rgName
    $subnet = Get-AzureRmVirtualNetworkSubnetConfig `
      -Name DMZ `
      -VirtualNetwork $virtualNetwork
    $nic = New-AzureRmNetworkInterface `
      -ResourceGroupName $rgName `
      -Location $location `
      -Name 'myNic-Nva' `
      -SubnetId $subnet.Id `
      -PrivateIpAddress 10.0.2.4 `
      -EnableIPForwarding `
      -PublicIpAddressId $Pip.Id 
    ```

7. Create the NVA virtual machine. The NVA can be a virtual machine running the Linux or Windows operating system. To create the virtual machine, copy the script for either operating system and paste it into your PowerShell session. If creating a Windows VM, paste the script into a text editor, change the "value" for the $cred variable, then paste the modified text into your PowerShell session:

    **Linux**

    ```azurepowershell-interactive
    # Define the admin user name and a blank password.
    $securePassword = ConvertTo-SecureString ' ' -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential ("azureuser", $securePassword)

    # Define the virtual machine configuration.
    $vmConfig = New-AzureRmVMConfig `
      -VMName 'myVm-Nva' `
      -VMSize Standard_DS2 | `
      Set-AzureRmVMOperatingSystem -Linux `
      -ComputerName 'myVm-Nva' `
      -Credential $cred -DisablePasswordAuthentication| `
      Set-AzureRmVMSourceImage `
      -PublisherName Canonical `
      -Offer UbuntuServer `
      -Skus 14.04.2-LTS `
      -Version latest | `
      Add-AzureRmVMNetworkInterface -Id $nic.Id

    # Configure SSH Keys
    $sshPublicKey = Get-Content "$env:USERPROFILE\.ssh\id_rsa.pub"
    Add-AzureRmVMSshPublicKey -VM $vmconfig -KeyData $sshPublicKey -Path "/home/azureuser/.ssh/authorized_keys"    

    # Create the virtual machine
    $vm = New-AzureRmVM `
      -ResourceGroupName $rgName `
      -Location $location `
      -VM $vmConfig
    ```

    **Windows**

    ```azurepowershell-interactive
    # Create user object
    $cred = Get-Credential -Message "Enter a username and password for the virtual machine."

    $vmConfig = New-AzureRmVMConfig `
      -VMName 'myVm-Nva' `
      -VMSize Standard_DS2 | `
      Set-AzureRmVMOperatingSystem -Windows `
      -ComputerName 'myVm-Nva' `
      -Credential $cred | `
      Set-AzureRmVMSourceImage `
      -PublisherName MicrosoftWindowsServer `
      -Offer WindowsServer `
      -Skus 2016-Datacenter `
      -Version latest | `
      Add-AzureRmVMNetworkInterface -Id $nic.Id
    
    $vm = New-AzureRmVM `
      -ResourceGroupName $rgName `
      -Location $location `
      -VM $vmConfig
    ```

8. By default, Azure routes traffic between all subnets within a virtual network. Create a route to change Azure's default routing so that traffic from the *Public* subnet is routed to the NVA, instead of directly to the *Private* subnet.

    ```azurepowershell-interactive    
    $routePrivate = New-AzureRmRouteConfig `
      -Name 'ToPrivateSubnet' `
      -AddressPrefix 10.0.1.0/24 `
      -NextHopType VirtualAppliance `
      -NextHopIpAddress $nic.IpConfigurations[0].PrivateIpAddress
    ```

9. Create a route table for the *Public* subnet.

    ```azurepowershell-interactive
    $routeTablePublic = New-AzureRmRouteTable `
      -Name 'myRouteTable-Public' `
      -ResourceGroupName $rgName `
      -location $location `
      -Route $routePrivate
    ```
    
10. Associate the route table to the Public subnet. Associating a route table to a subnet causes Azure to route all outbound traffic from the subnet according to the routes in the route table. A route table can be associated to zero or multiple subnets, whereas a subnet can have zero, or one route table associated to it.

    ```azurepowershell-interactive
    Set-AzureRmVirtualNetworkSubnetConfig `
      -VirtualNetwork $virtualNetwork `
      -Name 'Public' `
      -AddressPrefix 10.0.0.0/24 `
      -RouteTable $routeTablePublic | `
    Set-AzureRmVirtualNetwork
    ```

11. Create a route for traffic from the *Private* subnet to the *Public* subnet through the NVA virtual machine.

    ```azurepowershell-interactive    
    $routePublic = New-AzureRmRouteConfig `
      -Name 'ToPublicSubnet' `
      -AddressPrefix 10.0.0.0/24 `
      -NextHopType VirtualAppliance `
      -NextHopIpAddress $nic.IpConfigurations[0].PrivateIpAddress
    ```

12. Create the route table for the *Private* subnet.

    ```azurepowershell-interactive
    $routeTablePrivate = New-AzureRmRouteTable `
      -Name 'myRouteTable-Private' `
      -ResourceGroupName $rgName `
      -location $location `
      -Route $routePublic
    ```
      
13. Associate the route table to the *Private* subnet.

    ```azurepowershell-interactive
    Set-AzureRmVirtualNetworkSubnetConfig `
      -VirtualNetwork $virtualNetwork `
      -Name 'Private' `
      -AddressPrefix 10.0.1.0/24 `
      -RouteTable $routeTablePrivate | `
    Set-AzureRmVirtualNetwork
    ```
    
14. **Optional:** Create a virtual machine in the Public and Private subnets and validate that communication between the virtual machines is routed through the network virtual appliance by completing the steps in [Validate routing](#validate-routing).
15. **Optional**: To delete the resources that you create in this tutorial, complete the steps in [Delete resources](#delete-resources).

## Validate routing

1. If you haven't already, complete the steps in [Create routes and network virtual appliance](#create-routes-and-network-virtual-appliance).
2. Click the **Try it** button in the box that follows, which opens the Azure Cloud Shell. If prompted, log in to Azure using your [Azure account](../azure-glossary-cloud-terminology.md?toc=%2fazure%2fvirtual-network%2ftoc.json#account). If you don't have an Azure account, you can sign up for a [free trial](https://azure.microsoft.com/offers/ms-azr-0044p). The Azure Cloud Shell is a free bash shell with the Azure command-line interface preinstalled. 

    The following scripts create two virtual machines, one in the *Public* subnet, and one in the *Private* subnet. The scripts also enable IP forwarding for the network interface within the operating system of the NVA to enable the operating system to route traffic through the network interface. A production NVA typically inspects the traffic before routing it, but in this tutorial, the simple NVA just routes the traffic without inspecting it. 

    Click the **Copy** button in the **Linux** or **Windows** scripts that follow and paste the script contents into a text editor. Change the password for the *adminPassword* variable, then paste the script into the Azure Cloud Shell. Run the script for the operating system you selected when you created the network virtual appliance in step 7 of [Create routes and network virtual appliance](#create-routes-and-network-virtual-appliance). 

    **Linux**

    ```azurecli-interactive
    #!/bin/bash

    #Set variables used in the script.
    rgName="myResourceGroup"
    location="eastus"
    adminPassword=ChangeToYourPassword
    
    # Create a virtual machine in the Public subnet.
    az vm create \
      --resource-group $rgName \
      --name myVm-Public \
      --image UbuntuLTS \
      --vnet-name myVnet \
      --subnet Public \
      --public-ip-address myPublicIp-Public \
      --admin-username azureuser \
      --admin-password $adminPassword

    # Create a virtual machine in the Private subnet.
    az vm create \
      --resource-group $rgName \
      --name myVm-Private \
      --image UbuntuLTS \
      --vnet-name myVnet \
      --subnet Private \
      --public-ip-address myPublicIp-Private \
      --admin-username azureuser \
      --admin-password $adminPassword

    # Enable IP forwarding for the network interface in the NVA virtual machine's operating system.    
    az vm extension set \
      --resource-group $rgName \
      --vm-name myVm-Nva \
      --name customScript \
      --publisher Microsoft.Azure.Extensions \
      --settings '{"commandToExecute":"sudo sysctl -w net.ipv4.ip_forward=1"}'
    ```

    **Windows**

    ```azurecli-interactive

    #!/bin/bash
    #Set variables used in the script.
    rgName="myResourceGroup"
    location="eastus"
    adminPassword=ChangeToYourPassword
    
    # Create a virtual machine in the Public subnet.
    az vm create \
      --resource-group $rgName \
      --name myVm-Public \
      --image win2016datacenter \
      --vnet-name myVnet \
      --subnet Public \
      --public-ip-address myPublicIp-Public \
      --admin-username azureuser \
      --admin-password $adminPassword

    # Allow pings through the Windows Firewall.
    az vm extension set \
      --publisher Microsoft.Compute \
      --version 1.9 \
      --name CustomScriptExtension \
      --vm-name myVm-Public \
      --resource-group $rgName \
      --settings '{"commandToExecute":"netsh advfirewall firewall add rule name=Allow-ping protocol=icmpv4 dir=in action=allow"}'

    # Create a virtual machine in the Private subnet.
    az vm create \
      --resource-group $rgName \
      --name myVm-Private \
      --image win2016datacenter \
      --vnet-name myVnet \
      --subnet Private \
      --public-ip-address myPublicIp-Private \
      --admin-username azureuser \
      --admin-password $adminPassword

    # Allow pings through the Windows Firewall.
    az vm extension set \
      --publisher Microsoft.Compute \
      --version 1.9 \
      --name CustomScriptExtension \
      --vm-name myVm-Private \
      --resource-group $rgName \
      --settings '{"commandToExecute":"netsh advfirewall firewall add rule name=Allow-ping protocol=icmpv4 dir=in action=allow"}'

    # Enable IP forwarding for the network interface in the NVA virtual machine's operating system.
    az vm extension set \
      --publisher Microsoft.Compute \
      --version 1.9 \
      --name CustomScriptExtension \
      --vm-name myVm-Nva \
      --resource-group $rgName \
      --settings '{"commandToExecute":"powershell.exe Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters -Name IpEnableRouter -Value 1"}'

    # Restart the NVA virtual machine.
    az vm extension set \
      --publisher Microsoft.Compute \
      --version 1.9 \
      --name CustomScriptExtension \
      --vm-name myVm-Nva \
      --resource-group $rgName \
      --settings '{"commandToExecute":"powershell.exe Restart-Computer -ComputerName myVm-Nva -Force"}'
    ```

3. Validate communication between the virtual machines in the Public and Private subnets. 

    - Open an [SSH](../virtual-machines/linux/tutorial-manage-vm.md?toc=%2fazure%2fvirtual-network%2ftoc.json#connect-to-vm) (Linux) or [Remote Desktop](../virtual-machines/windows/tutorial-manage-vm.md?toc=%2fazure%2fvirtual-network%2ftoc.json#connect-to-vm) (Windows) connection to the public IP address of the *myVm-Public* virtual machine.
    - From a command prompt on the *myVm-Public* virtual machine, enter `ping myVm-Private`. You receive replies because the NVA routes the traffic from the public to the private subnet.
    - From the *myVm-Public* virtual machine, run a trace route between the virtual machines in the public and private subnets. Enter the appropriate command that follows, depending upon which operating system you installed in the virtual machines in the Public and Private subnets:
        - **Windows**: From a command prompt, run the `tracert myvm-private` command.
        - **Ubuntu**: Run the `tracepath myvm-private` command.
      Traffic passes through 10.0.2.4 (the NVA) before reaching 10.0.1.4 (the virtual machine in the Private subnet). 
    - Complete the previous steps by connecting to the *myVm-Private* virtual machine and pinging the *myVm-Public* virtual machine. The trace route shows communication traveling through 10.0.2.4 before reaching 10.0.0.4 (the virtual machine in the Public subnet).
    - **Optionally**: Use the next hop capability of Azure Network Watcher to validate the next hop between two virtual machines within Azure. Before using Network Watcher, you must first [create an Azure Network Watcher instance](../network-watcher/network-watcher-create.md?toc=%2fazure%2fvirtual-network%2ftoc.json) for the region you want to use it in. In this tutorial, the US East region is used. Once you've enabled a Network Watcher instance for the region, enter the following command to see the next hop information between the virtual machines in the Public and Private subnets:
     
        ```azurecli-interactive
        az network watcher show-next-hop --resource-group myResourceGroup --vm myVm-Public --source-ip 10.0.0.4 --dest-ip 10.0.1.4
        ```

       The output returns *10.0.2.4* as the **nextHopIpAddress** and *VirtualAppliance* as the **nextHopType**.

> [!NOTE]
> To illustrate the concepts in this tutorial, public IP addresses are assigned to the virtual machines in the Public and Private subnets, and all network port access is enabled within Azure for both virtual machines. When creating virtual machines for production use, you may not assign public IP addresses to them and may filter network traffic to the Private subnet by deploying a network virtual appliance in front of it, or by assigning a network security group to the subnets, network interface, or both. To learn more about network security groups, see [Network security groups](virtual-networks-nsg.md).

## Create a virtual network

This tutorial requires an existing virtual network with two subnets. Click the **Try it** button in the box that follows, to quickly create a virtual network. Clicking the **Try it** button opens the [Azure Cloud Shell](../cloud-shell/overview.md?toc=%2fazure%2fvirtual-network%2ftoc.json). Though the Cloud Shell runs PowerShell or a Bash shell, in this section, the Bash shell is used to create the virtual network. The Bash shell has the Azure command-line interface installed. If prompted by the Cloud Shell, log in to Azure using your [Azure account](../azure-glossary-cloud-terminology.md?toc=%2fazure%2fvirtual-network%2ftoc.json#account). If you don't have an Azure account, you can sign up for a [free trial](https://azure.microsoft.com/offers/ms-azr-0044p). To create the virtual network used in this tutorial, click the **Copy** button in the following box, then paste the script into the Azure Cloud Shell:

```azurecli-interactive
#!/bin/bash

#Set variables used in the script.
rgName="myResourceGroup"
location="eastus"

# Create a resource group.
az group create \
  --name $rgName \
  --location $location

# Create a virtual network with one subnet named Public.
az network vnet create \
  --name myVnet \
  --resource-group $rgName \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name Public \
  --subnet-prefix 10.0.0.0/24

# Create an additional subnet named Private in the virtual network.
az network vnet subnet create \
  --name Private \
  --address-prefix 10.0.1.0/24 \
  --vnet-name myVnet \
  --resource-group $rgName
```

To learn more about how to use the portal, PowerShell, or an Azure Resource Manager template to create a virtual network, see [Create a virtual network](virtual-networks-create-vnet-arm-pportal.md).

## Delete resources

When you finish this tutorial, you might want to delete the resources that you created, so that you don't incur usage charges. Deleting a resource group also deletes all resources that are in the resource group. In PowerShell, enter the following command:

```azurepowershell-interactive
Remove-AzureRmResourceGroup -Name myResourceGroup -Force
```

## Next steps

- Create a [highly available network virtual appliance](/azure/architecture/reference-architectures/dmz/nva-ha?toc=%2fazure%2fvirtual-network%2ftoc.json).
- Network virtual appliances often have multiple network interfaces and IP addresses assigned to them. Learn how to [add network interfaces to an existing virtual machine](virtual-network-network-interface-vm.md#vm-add-nic) and [add IP addresses to an existing network interface](virtual-network-network-interface-addresses.md#add-ip-addresses). Though all virtual machine sizes can have at least two network interfaces attached to them, each virtual machine size supports a maximum number of network interfaces. To learn how many network interfaces each virtual machine size supports, see [Windows](../virtual-machines/windows/sizes.md?toc=%2Fazure%2Fvirtual-network%2Ftoc.json) and [Linux](../virtual-machines/linux/sizes.md?toc=%2fazure%2fvirtual-network%2ftoc.json) virtual machine sizes. 
- Create a user-defined route to force tunnel traffic on-premises through a [site-to-site VPN connection](../vpn-gateway/vpn-gateway-forced-tunneling-rm.md?toc=%2fazure%2fvirtual-network%2ftoc.json).
