---
Author: "Andy Clifton"
title: "*Default Outbound Internet Access*"
description: "Default Outbound Internet Access is no longer allowed by default."
date: 2025-08-12
tags: ["Networking"]
thumbnail: "/img/outbound.jpeg"
---



![alt](/Images/Outbound/Outbound.Jpeg)
 
# Default Outbound Internet Access is being retired

### ==> Disclaimer work in progress. <==

## Introduction
Until now, Microsoft gave every new Virtual Network outbound internet access by default.
This happened automatically through a default route to the internet, without you having to do anything for it.
This default setting provided internet access automatically without you having to configure it explicitly.
That seemed convenient, but it came with risks: uncontrolled traffic, hard to monitor, and vulnerable to data leaks.
Security teams had less grip on outbound flows, which did not fit within Zero Trust principles.
From now on you have to arrange outbound access explicitly yourself, via a NAT Gateway or other solutions.

![alt](/Images/Outbound/network.jpeg)
<ins>What are the consequences?</ins>

Until now, Azure automatically gave you internet access when you created a new Virtual Network.
Convenient, right? You did not have to do anything extra. Your VM could simply reach the internet.
But as of 30 September that is a thing of the past. Microsoft is turning this automatic access off.

<ins>What will you notice?</ins>

- New Virtual Networks no longer have automatic internet access.  
- Applications that depend on outbound internet must be configured explicitly.
- You have to set up a NAT Gateway or another outbound solution yourself.
- Deployment scripts and templates can fail if they assume default internet access.
- Monitoring and logging become more important for managing outbound traffic.
- Security gets tighter: no more uncontrolled connections.
- You get more control, but also more responsibility.
- Costs can rise due to the use of NAT Gateways.
- Legacy architectures may need to be adjusted.
- It forces teams to deal more consciously with network security and outbound flows.

**For existing VNets nothing changes at this moment**

## What solutions are there to arrange this properly?

![alt](/Images/Outbound/optie.jpeg)


**Azure NAT Gateway**

This is the solution Microsoft recommends for most environments, because:

*Improved security*

NAT Gateway provides controlled outbound internet access, allowing you to determine exactly which resources have access to the internet. This rules out uncontrolled connections and fits better with Zero Trust principles.

*Logging and monitoring*

With NAT Gateway you can monitor and log outbound traffic more effectively. This makes it easier to detect and audit suspicious activity.

*Scalability*

NAT Gateway is designed for high scalability and supports thousands of concurrent connections, which is essential for large workloads.

*Static IP addresses*

You can assign one or more static public IP addresses to your outbound traffic, which makes managing firewall rules and access policies easier.

*Cost management*

NAT Gateway makes it possible to centralise outbound traffic, so you can optimise costs and avoid unexpected expenses caused by uncontrolled outbound flows.


**Azure Load Balancer (Outbound Rules)**

Azure Load Balancer can also be used to arrange outbound internet access, especially in scenarios where you already use a Load Balancer for inbound traffic. Why is this a good replacement?

*Flexible outbound configuration*

With outbound rules you can determine exactly which resources get outbound internet access, and how the traffic is handled.

*Static IP addresses*

You can associate public IP addresses with your outbound traffic, so you always know from which IP your resources reach the internet.

*Scalability*

Azure Load Balancer supports large numbers of concurrent connections and is suitable for both small and large workloads.

*Integration with existing infrastructure*

If you already use a Load Balancer for inbound traffic, you can easily add outbound access without extra components.

*Cost savings*

For some scenarios, Load Balancer can be cheaper than NAT Gateway, especially for smaller workloads or if you already have a Load Balancer running.

**Azure Firewall**

Azure Firewall is an advanced solution for outbound internet access, particularly suited to environments where security and compliance are central. Why is this a good replacement?

*Advanced security*

Azure Firewall offers deep inspection of outbound traffic, including filtering at the application and network layer. You can set strict rules for allowed and blocked connections.

*Logging, monitoring and threat intelligence*

Firewall logs are very detailed and can be integrated with Azure Monitor and Sentinel. You gain insight into all outbound flows and can detect threats quickly.

*Scalability*

Azure Firewall scales automatically with your workload and supports large numbers of concurrent connections.

*Static and multiple public IP addresses*

You can assign one or more public IP addresses to outbound traffic, which is useful for access management and compliance.

*Centralisation and segmentation*

With Azure Firewall you can manage outbound access centrally for multiple subnets and VNets, and apply segmentation for extra security.
The downside of Azure Firewall is that it is, relatively speaking, a fairly expensive solution.

**Public IP on the VM**

Assigning a Public IP address directly to a VM is a simple way to arrange outbound internet access. Why can this be an alternative?

*Direct internet access*

The VM has direct access to the internet without intermediate components, which keeps configuration simple.

*Static IP address*

You can assign a static Public IP address, so you always know from which IP your VM reaches the internet.

*Simple troubleshooting*

Problems with outbound traffic are easier to trace because the traffic comes directly from the VM.

*No extra costs for NAT or Load Balancer*

You only pay for the Public IP address, not for extra network components.

*Suitable for small workloads*

For test or development environments this can be a quick and cheap solution.

Note: this option is less secure and less scalable than NAT Gateway, Load Balancer or Firewall. Only use it when security does not play a major role.

## How can we check which machines depend on outbound access

It is important to know which virtual machines in your environment depend on outbound internet access, so you can take measures in time. We use PowerShell for this. With PowerShell you can, for example, query which VMs have a public IP address. By collecting this information, you can determine specifically which workloads need extra attention when phasing out default outbound access.  
Run the PowerShell script below to get an output of the servers you need to adjust.

````
# Login to Azure
Connect-AzAccount

# Select subscription
$subscriptionId = "<your-subscription-id>"
Select-AzSubscription -SubscriptionId $subscriptionId

# Results array
$results = @()

# Get all VMs
$vms = Get-AzVM

foreach ($vm in $vms) {
    $vmName = $vm.Name
    $rgName = $vm.ResourceGroupName
    $hasPublicIp = $false
    $hasOutboundNSG = $false
    $hasInternetRoute = $false

    # Get NIC
    $nicId = $vm.NetworkProfile.NetworkInterfaces[0].Id
    $nicName = ($nicId -split "/")[-1]
    $nic = Get-AzNetworkInterface -Name $nicName -ResourceGroupName $rgName

    # Check for public IP
    foreach ($ipConfig in $nic.IpConfigurations) {
        if ($ipConfig.PublicIpAddress) {
            $hasPublicIp = $true
            $publicIpName = ($ipConfig.PublicIpAddress.Id -split "/")[-1]
            $publicIpRg = ($ipConfig.PublicIpAddress.Id -split "/")[4]
            $publicIp = Get-AzPublicIpAddress -Name $publicIpName -ResourceGroupName $publicIpRg
        }
    }


    # Check NSG outbound rules
    if ($nic.NetworkSecurityGroup) {
        $nsgName = ($nic.NetworkSecurityGroup.Id -split "/")[-1]
        $nsgRg = ($nic.NetworkSecurityGroup.Id -split "/")[4]
        $nsg = Get-AzNetworkSecurityGroup -Name $nsgName -ResourceGroupName $nsgRg
        $outboundRules = $nsg.SecurityRules | Where-Object { $_.Direction -eq "Outbound" -and $_.Access -eq "Allow" }
        if ($outboundRules.Count -gt 0) {
            $hasOutboundNSG = $true
        }
    }


    # Check route table
    $subnetId = $nic.IpConfigurations[0].Subnet.Id
    $vnetRg = ($subnetId -split "/")[4]
    $vnetName = ($subnetId -split "/")[8]
    $vnet = Get-AzVirtualNetwork -Name $vnetName -ResourceGroupName $vnetRg
    $subnetName = ($subnetId -split "/")[-1]
    $subnet = $vnet.Subnets | Where-Object { $_.Name -eq $subnetName }

    if ($subnet.RouteTable) {
        $routeTable = Get-AzRouteTable -ResourceId $subnet.RouteTable.Id
        $internetRoute = $routeTable.Routes | Where-Object { $_.AddressPrefix -eq "0.0.0.0/0" -and $_.NextHopType -eq "Internet" }
        if ($internetRoute) {
            $hasInternetRoute = $true
        }
    }

    # Output to console
    Write-Host "VM: $vmName"
    Write-Host "  ✅ Public IP: $hasPublicIp"
    Write-Host "  🔄 NSG Outbound Access: $hasOutboundNSG"
    Write-Host "  🌐 Internet Route: $hasInternetRoute"
    Write-Host "---------------------------------------------"

    # Add to results
    $results += [PSCustomObject]@{
        VMName           = $vmName
        ResourceGroup    = $rgName
        PublicIP         = $hasPublicIp
        NSGOutboundAllow = $hasOutboundNSG
        InternetRoute    = $hasInternetRoute
    }
}

# Export to CSV
$results | Export-Csv -Path "OutboundAccessReport.csv" -NoTypeInformation
Write-Host "✅ Rapport opgeslagen als 'OutboundAccessReport.csv'"

````

We have now investigated which machines have a Public IP, NSG rules or an Internet Route.
Now we need to implement the chosen option. We will discuss creating an Azure Firewall another time,
adding a Public IP speaks for itself, so we are now going to create an Azure NAT Gateway.

## Creating an Azure NAT Gateway.
![Image](/Images/Outbound/Nat.png)

Azure NAT Gateway is the recommended solution for providing secure and scalable outbound internet access for your virtual machines and other resources in Azure. With NAT Gateway you can centralise and manage outbound traffic, so you know exactly which IP addresses are used for internet access. This makes it easier to apply firewall rules, set up monitoring and secure your network architecture. In addition, NAT Gateway provides high availability and supports thousands of concurrent connections, making it suitable for both small and large environments. Below you can read how to create and configure a NAT Gateway.
But how do you configure this?

Sign in to the Azure portal and click **Create a resource**.

![Image](/Images/Outbound/NatGateway.png)

Enter **NAT Gateway** and click **Create**.

Fill in the fields and click **Next**
~~~
Subscription = Subscription that connectivity is attached to
Resource Group = RG for Connectivity

NAT Gateway Name = Name
Region = Region where you deploy everything. (West Europe in this case)
Availability Zone = Indicate whether you want to use this in case of issues in the datacenter
TCP Idle timeout = Time between 4 and 120 minutes before a timeout occurs and the connection is closed.
~~~

![Image](/Images/Outbound/NatGateway1.png)


Now we need to configure the outbound addresses.

First click **Create a new public IP address**

~~~
Name = name for the public IP.
SKU = Basic
Assignment = Static
~~~

Click **Ok**

![Image](/Images/Outbound/pip.png)
Next we configure the prefix (in other words, how many Public IPs can still be created in this NAT Gateway at most).

Click **Create a new public IP prefix**

~~~
Name = name for the public prefix.
Prefix Size= Pick the size you need. In this test I pick a /31 because I do not need more than 2 external addresses.
~~~
Click **Ok**

![Image](/Images/Outbound/pip2.png)

Click **Next**

We have now arrived at configuring the subnets that will use this NAT.
So select the Virtual network and then the subnets.

![Image](/Images/Outbound/NAT1.png)

Click **Review+Create** and then **Create**

![Image](/Images/Outbound/passed.png)

We can also build this with PowerShell, which you do with the PowerShell script below.

````
# Set the variables for the NAT Gateway.
$rg = "RG-Andy-Test2"
$Location = "Westeurope"
$sku = "Standard"
$PublicIpname = "pip"
$Publicprefixname = "PIPPrefix"
$NatGatewayname="NATGateway"


#create Rsource group
New-AzResourceGroup -Name $rg -Location $Location 

#create Standard SKUP public IP
$publicIP = New-AzPublicIpAddress -Name $PublicIpname -ResourceGroupName $rg -AllocationMethod Static -Location $Location -Sku $sku

#create  IP prefix
$publicIPPrefix = New-AzPublicIpPrefix -Name $Publicprefixname -ResourceGroupName $rg -Location $Location -PrefixLength 29

#Create NAT gateway
$natGateway = New-AzNatGateway -Name $NatGatewayname -ResourceGroupName $rg -PublicIpAddress $publicIP -PublicIpPrefix $publicIPPrefix -Location $Location -Sku $sku -IdleTimeoutInMinutes 4
$natGateway  | Select-Object Name, ResourceGroupName, IdleTimeoutInMinutes , SKuText | Format-table -autosize –wrap

$virtualNetwork = Get-AzVirtualNetwork | Out-GridView -PassThru -Title "Kies je VNET"
$NATSubnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $virtualNetwork | Out-GridView -PassThru -Title "Kies de Subnets"

````

The NAT Gateway is now deployed and configured; the only thing left to do is sign in to the VM and test whether outbound traffic works.

## Want to ensure a uniform approach right now? Then tackle the default internet access in existing VNets straight away.

This is how you do it:

🔒 *Block unwanted internet traffic*
Create an outbound NSG rule that blocks all traffic to the internet, except via the route you have chosen.

🚧 *Deliberately steer the traffic in the right direction*
Explicitly associate a NAT Gateway or Firewall with your VNet and make sure other routes disappear from the picture.

That way you keep a firm grip on things and prevent traffic from quietly slipping out through the back door.

