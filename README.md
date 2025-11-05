# A little big home lab Kubernetes cluster
A home lab Kubernetes cluster using **AKS Arc** on **Azure Local**, built with budget hardware yet having an enterprise-grade feature set.


Motivated by all the nice setups from [Lior Kamrat](https://www.linkedin.com/in/liorkamrat/recent-activity/all/), I want a home lab as well to run tools, gameservers, my own apps and test a few things from the [CNCF landscape](https://landscape.cncf.io/) in my local network. But there is not enough budget for AI workloads 😜. lets skipt that for now.  
*Remarks - this will not be a comparsion about the best Kubernetes cluster or Virtual Machines home lab setup across all available offerings. I'll focus on [Azure Local](https://azure.microsoft.com/en-us/products/local/) and use [Unify](https://www.ui.com/) hardware -simply because I know it very well, it will be quicker for me than starting from scratch.*

Ok - whats the goal: Create an Azure Local, run an AKS Arc Kubernetes cluster on it, and avoid having anything to do with an Active Directory Domain. I'll use the "AD less" deployment - [Deploy Azure Local using local identity with Azure Key Vault](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-local-identity-with-key-vault?view=azloc-2508) and create a [Kubernetes cluster](https://learn.microsoft.com/en-us/azure/aks/aksarc/aks-whats-new-local) on it. There is no internet proxy to overcome at home, so I'll skip using [Arc Gateway](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-azure-arc-gateway-overview?view=azloc-2508&tabs=portal#how-it-works) for now. I'll test offline mode and Arc Gateway in a second step.  
*Note - Azure Local using local identity with Azure Key Vault is still flagged as preview.*

### The hardware:  
The much I like [Ben's home lab](https://bcthomas.com/) - there is no rack space at home 🤷‍♂️ - better to go the other direction: affordable hardware as small and silent as possbile. Something on the sweetspot of price/value, with a low power consumption. A thank you here to [Stefan Denninger](https://www.linkedin.com/in/stefan-denninger/) suggesting me using a Minisforum [MS-01](https://store.minisforum.com/products/minisforum-ms-01) - that little thing fulfills all low capacity Azure Local single node [requirements](https://learn.microsoft.com/en-us/azure/azure-local/concepts/system-requirements-small-23h2?view=azloc-2508#device-requirements) *except the ECC memory!* but more about it later. I use for this project:
- [x] [MS-01 Barebone with Intel i9-12900H](https://minisforumpc.eu/products/ms-01?_pos=1&_psq=MS&_ss=e&_v=1.0&variant=42097212489911)
- [x] Kingston FURY SO-DIMM 64GB KIT DDR5 4800MHz CL38 - 2 modules with 32 GB
- [x] 1x 1TB Samsung 990 EVO Plus NVMe SSD for the OS
- [x] 2x 2TB Samsung 990 EVO Plus NVMe SSDs for data  
By using the Intel I226V RJ45 Ethernet adapter the Azure Local cluster deployment checks complained about 'Network requirements not met - Cannot find valid RSS property, adapter need support RSS. Use Get-NetAdapterRss to verify it. I use the Intel X710 adapater instead, it has [RSS](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/introduction-to-receive-side-scaling) support, and that's why there is a  
- [x] 1x 10G SFP+ to RJ45 Copper Module
on the hardware list

### Network:
I run a [UniFi Dream Machine Special Edition](https://techspecs.ui.com/unifi/cloud-gateways/udm-se) with a public static IP, Wirguard VPN to connect to my home network from everywhere and fullfilling all requirements for any kind of home lab you can imagine. Except the fact that I'm always running out of switch ports 🙄

You need a [DNS server](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-local-identity-with-key-vault#configure-dns-server-for-azure-local). One option is to use the DNS server from the Dream Machine, but for this setup I decided to use one of my test domains and the DNS server from the hosting provider. There are two DNS entries required:
- [x] the host machine: `hostname.mydomain.com` -> 192.168.1.1
- [x] the Azure Local cluster object: `azlmyname.mydomain.com` -> 192.168.2.1 (the first IP of the 6 required one for Azure Local)


### The Machine:
First thing to do - connect a monitor, mouse, keyboard and set the primary display from 'auto' to 'HG' in Bios, otherwise I always had a black screen after a reboot. [Looks like it's not an issue of my monitor](https://spaceterran.com/posts/step-by-step-guide-enabling-intel-vpro-on-your-minisforum-ms-01-bios/#disabling-aspm)

I wanted to use Intel vPro / AMT - there are very good guides from [https://www.youtube.com/@RaspberryPiCloud](https://www.youtube.com/watch?v=cxWl3gyl-qU) or [Space Terran](https://spaceterran.com/posts/step-by-step-guide-enabling-intel-vpro-on-your-minisforum-ms-01-bios/#configuring-mebx) available. 
My advise - skip it. Even after finally figuring out the complex password policy for this in the BIOS, I failed to make it work within 2 hours and decided to use RemoteDesktop and PS remoting instead.
The few times you need it - save your lifetime - connecting a monitor mouse and keyboard takes a few minutes.

Alright - lets [get Auzre Local](https://portal.azure.com/#view/Microsoft_Azure_ArcCenterUX/ArcCenterMenuBlade/~/hciGetStarted) and create a bootable usb stick with [Rufus](https://rufus.ie/).
Meanwhile, [perpare your Azure subscription](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-arc-register-server-permissions):  

- [x] Register Resource Providers on the subscription
- [x] Create a resource group
- [x] Set required permissions on the subscription
- [x] Set required permissions on the resource group

Logon to the machine, [Sconfig](https://learn.microsoft.com/en-us/windows-server/administration/server-core/server-core-sconfig#start-sconfig) shows up. Enable Remote Desktop (alternative over Powershell with `Enable-ASRemoteDesktop`) and lets do the infra setup.


### Deploy Azure Local using local identity with Azure Key Vault
[docs are here](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-local-identity-with-key-vault)  

#### 1. net adapter
DHCP is not supported, lets configure the Intel X710 adapter IP (RSS support!). I use IP range 192.168.0.1/20 (192.168.0.0 - 192.168.15.255)
``` 
192.168.1.1 / 255.255.240.0
Gateway: 192.168.0.1
DNS: 192.168.0.1
```

#### 2. hostname
Set the hostname to `myhostname` with Sconfig. Add a DNS entry `myhostname.mydomain.com` at your local DNS server / domain hoster - in my case the DNS server of the hosting provider. Verify with
``` 
nslookup myhostname
nslookup myhostname.mydomain.com
```

#### 3. set DNS zone
Set registry key with the zone name/FQDN on each node 
```
$zoneName = "mydomain.com" 
$RegistryPath = 'HKLM:\SYSTEM\CurrentControlSet\services\Tcpip\Parameters' 
Set-ItemProperty -Path $RegistryPath -Name 'Domain' -Value $zoneName
```
and restart the machine with
```
Restart-Computer
```

#### 4. deployment user
Create local admin user for Azure Local deployment
```
$Password = Read-Host -AsSecureString
$params = @{
    Name        = 'myuser'
    Password    = $Password
    FullName    = 'Firstname Lastname'
    Description = 'local setup'
}
New-LocalUser @params
Add-LocalGroupMember -Group "Administrators" -Member "gstoifl"
```

#### 5. no ECC mmeory
I don't have ECC memory modules and need to disable the check, I have luck, there is a feature flag to disable the ECC requirement check.
This will skip ECC tests, documented in AzStackHciConnectivity\AzStackHci.Connectivity.psm1 and AzStackHciHardware\AzStackHci.Hardware.psm1  
```
New-Item -Path "C:\Program Files\WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker\ExcludeTests.txt" -ItemType File
Set-Content -Path "C:\Program Files\WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker\ExcludeTests.txt" -Value "Test-MemoryProperties"
```

#### 6. Arc onboarding
Register the machine with Azure Arc: [Onboard by script](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-without-azure-arc-gateway?view=azloc-2509&tabs=script&pivots=register-proxy#step-1-review-script-parameters)
```
$Tenant = <"my TenantID">
$Subscription = "<my subscription">
$RG = "<my resource group>"
$Region = "<my region>"
$Tenant = "<my tenant>"
Invoke-AzStackHciArcInitialization -TenantId $Tenant -SubscriptionID $Subscription -ResourceGroup $RG -Region $Region -Cloud "AzureCloud"
```

#### Azure Portal - deploy Azure Local:  
Open Azure portal and create an Azure Local as [discribed here:](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-local-identity-with-key-vault#deploy-azure-local-via-the-portal-using-local-identity-with-key-vault)

Instance name `azlmyname`  

Group management and compute at network adapter 1 - 192.168.1.1   

Nodes and Instance IPs:  
```
192.168.2.1 - 192.168.2.254
255.255.240.0
Gateway: 192.168.0.1
DNS: 192.168.0.1
```
Create a DNS entry for the cluster object with the first IP from the range as [documented here](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-local-identity-with-key-vault#configure-dns-server-for-azure-local).  
A record `azlmyname.mydomain.com` to `192.168.2.1`  

#### In case of storage validation errors during Azure Local Setup:  
If validation fails at AzStackHci_Hardware_Test_PhysicalDisk - the data disks need to be clean and empty. If this command:
```
Get-PhysicalDisk | Format-Table PhysicalLocation, UniqueId, SerialNumber, CanPool, CannotPoolReason, BusType, MediaType, Size, FriendlyName
```
shows two SSDs with `CanPool` `false` and `CannotPoolReason` `In a Pool`, delete the SU1_Pool storage pool and reset the disk with:
```
Get-StoragePool
Remove-StoragePool -FriendlyName SU1_Pool
Reset-PhysicalDisk -UniqueId eui.0025385551A3CDA1
Reset-PhysicalDisk -UniqueId eui.0025385B4141DD51
```

## Setup AKS Arc

#### Azure Portal - create logical network:
Aks Arc requires a Azure Local logical network, I use the next 254 IPs in may range. If you have a bigger Azure Local cluster, use a bigger range. You can [click it](https://learn.microsoft.com/en-us/azure/aks/aksarc/aks-networks?tabs=azureportal) in Azure portal or use the [CLI](https://learn.microsoft.com/en-us/azure/aks/aksarc/aks-networks?tabs=azurecli)
```
192.168.16.0/20
192.168.3.1 - 192.168.3.254
255.255.240.0
Gateway: 192.168.0.1
DNS: 192.168.0.1
```

## Lets create a Kubernetes cluster:
Use [Azure Portal](https://learn.microsoft.com/en-us/azure/aks/aksarc/aks-create-clusters-portal#create-a-kubernetes-cluster) or the [CLI](https://learn.microsoft.com/en-us/azure/aks/aksarc/aks-create-clusters-portal#create-a-kubernetes-cluster):  

**Note** It's a good idea to create an ssh key upfront and keep the private key in your password manager - having the private key enables troubleshooting options later on. Create an SSH key as [documented here](https://learn.microsoft.com/en-us/azure/virtual-machines/ssh-keys-portal#generate-new-keys) and use it at cluster create in Azure Portal or CLI.
```
$name="<my cluster>"
$sub="<my subscription>"
$rg="<my resource group>"
$cl="<my custom location>"
$logicnet="<logical network created before>"
$controlplanesize="Standard_A2_v2"  # 2 core, 4GB mem  
$nodesize="Standard_A4_v2"  # 4 cores, 8GB mem  
az aksarc create -n $name -g $rg --custom-location $cl --vnet-ids $vnet --control-plane-vm-size $controlplanesize --node-vm-size $nodesize --ssh-key-value .\id_rsa.pub --verbose
```

### Add a loadbalancer
To reach deployments, lets add an loadbalancer. I'll use the next 254 IPs in the range 192.168.4.1 - 192.168.4.254
Enable it over [Azure Portal](https://learn.microsoft.com/en-us/azure/aks/aksarc/deploy-load-balancer-portal) or use the [CLI](https://learn.microsoft.com/en-us/azure/aks/aksarc/deploy-load-balancer-cli) 
```
$resource="subscriptions/$sub/resourceGroups/$rg/providers/Microsoft.Kubernetes/connectedClusters/$name"
$lbName="$name-lb"
$ipRange="192.168.4.1-192.168.4.254"

az k8s-runtime load-balancer enable --resource-uri $resource
az k8s-runtime load-balancer create --load-balancer-name $lbName --resource-uri $resource --addresses $ipRange --advertise-mode ARP
```

### Connect to the cluster - ready to use.
```
az aksarc get-credentials -g $rg -n $name= --admin
```