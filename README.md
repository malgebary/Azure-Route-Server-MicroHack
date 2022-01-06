# **Azure Route Server MicroHack**

# Contents

[Introduction](#introduction)

[Objectives](#objectives)

[Lab](#lab)

[Notes](#Notes)

[Scenario 1: Connect on-premises network to Azure and use ARS for route distribution](#Scenario-1-Connect-on-premises-network-to-Azure-and-use-ARS-for-route-distribution)

[Scenario 2: Route exchange between Virtual Network Gateway and CSR NVA](#Scenario-2-Route-exchange-between-Virtual-Network-Gateway-and-CSR-NVA)

[Scenario 3: Vnet peering and ARS](#Scenario-3-Vnet-peering-and-ARS)

[Scenario 4: Route server multi-region design with IPsec](#Scenario-4-Route-server-multi-region-design-with-Ipsec)

[Scenario 5: Route server multi-region design with Vnet peering](#Scenario-5-Route-server-multi-region-design-with-Vnet-peering)

[Scenario 6: On-premises internet breakout through Azure](#Scenario-6-On-premises-internet-breakout-through-Azure)


# Introduction

Azure Route Server ([ARS](https://docs.microsoft.com/en-us/azure/route-server/overview)) enables dynamic routing between Network Virtual Appliance (NVA) and Virtual Network (Vnet) by supporting Border Gateway Protocol (BGP). Before ARS, users had to create a User Defined Route (UDR) to NVA, and need to manually update the routing table on the NVA whenever Vnet addresses get updated. With the introduction of Azure Route Server, users can now inject a similar route without having to create the UDRs, or manually update the NVA with new Vnet prefixes.

ARS enables the NVA to advertise routes to other endpoints in the local/peered Vnet; as well as endpoints defined in on-premises networks connected via ExpressRoute or VPN.  Additionally, ARS allow transit between Virtual Network Gateways (ExpressRoute and VPN) which was not possible before. 

In this MicroHack we will explore some routing scenarios that shows how ARS can be utilized to simplify configuration, management, and deployment of the NVAs in the Virtual Network and how would that simplify routing within Azure and between Azure and on-premises.

# Objectives:

After completing this MicroHack you will:

- Know how to build Azure Route Server in a Vnet and connect it to an NVA and/or virtual network gateway.
- Understand the new routing capabilities introduced by ARS and how to interpret the route table after deploying the ARS in the Vnet.

# Lab:

Azure Cli will be used to build this Lab. the complete lab consist of:  

  1. Six Vnets: ***On-Prem-Vnet***, ***On-prem1-Vnet***, ***HUB-SCUS***, ***HUB-EastUS***, ***Spoke-Vnet***, and ***Spoke1-Vnet***.
  2. Three Virtual Network Gateways:***On-Prem-VNG***, ***On-Prem1-VNG***, and ***HUB-VNG***.
  3. Eight VMs: ***On-Prem-VM***, ***On-Prem1-VM***, ***HUB-VM***, ***HUB1-VM***, ***CSR***, ***CSR1***, ***Spoke-VM***, and ***Spoke1-VM***.
  4. Two Azure Route Severs: ***RouteServer***, and ***RouteServer1***.
	
  
The complete lab deployment will look like this:

![image](https://user-images.githubusercontent.com/78562461/147891873-c20be1a1-d7ac-4bed-9701-39820b153f9e.png)


# Notes:

- Log in to Azure Cloud Shell at https://shell.azure.com/ and select Bash.

  - If necessary select your target subscription:

	  az account set --subscription <Name or ID of subscription>

- All VMs have Internet access, no NSGs are used.

- Credentials:

	username: `azureuser`

	password: `Routeserver123`

	tunnel pre-shared key: `Routeserver`

- All resources will be launched in one resource group `Route-Server`.
	
- The scenarios in this lab are built sequntially, so that scenario 2 depends on scenario 1 configuration and so on. 	

## Scenario 1: Connect on-premises network to Azure and use ARS for route distribution

In this scenario you will connect on-premises network (***On-Prem-Vnet***) to Azure through NVA (CSR 1000V) deployed in ***HUB-SCUS*** Vnet, and we will observe the routing table before and after deploying ARS in the ***HUB-SCUS*** Vnet.

This scenario consist of:

1. ***On-prem-Vnet*** to simulate on-premises network, Vpn Gateway ***On-Prem-VNG*** to simulate gateway on the on-premises, and an VM ***On-Prem-VM***.
2. ***HUB-SCUS*** Vnet represent Azure side that has: NVA (***CSR***), VM (***HUB-VM***), and ARS (***RouteServer***).
3. NVA ***CSR*** with two Nics, one external (CSROutsideInterface) and one Internal (CSRInsideInterface), external interface will be used as the source for Ipsec tunnel, while internal interface will be used as BGP peer with ARS.
4. We will use loopback interface on the ***CSR*** (Loopback11) as update source for the BGP peering with ***On-Prem-VNG***, while will use the internal interface of the ***CSR*** as the BGP peer with ARS instances.
	

After deploying above, the diagram will look like following:

![image](https://user-images.githubusercontent.com/78562461/140993322-3bf5a79a-c6f3-4b9f-ad76-a82c1a3a78c5.png)

## Task1: Deploy On-Prem-Vnet and HUB-SCUS Vnet without the ARS 

- **Create the resource group:**

    az group create --name Route-Server --location centralus

- **Create the On-Premises Network:**

 **On-prem-Vnet**:

    az network vnet create --resource-group Route-Server --name On-Prem-Vnet --location northcentralus --address-prefixes 10.0.0.0/16 --subnet-name Subnet-1 --subnet-prefix 10.0.10.0/24 
    az network vnet subnet create --address-prefix 10.0.0.0/27 --name GatewaySubnet --resource-group Route-Server --vnet-name On-Prem-Vnet


**On-Prem-VM:**

    az network public-ip create --name On-PremVMIP --resource-group Route-Server --location northcentralus --allocation-method Dynamic
    
    az network nic create --resource-group Route-Server -n OnPrem-VMNIC --location northcentralus --subnet Subnet-1 --private-ip-address 10.0.10.4 --vnet-name On-Prem-Vnet --public-ip-address  On-PremVMIP
    
    az vm create -n On-Prem-VM -g Route-Server --image UbuntuLTS --admin-username azureuser --admin-password Routeserver123 --size Standard_B1ls --location northcentralus --private-ip-address 10.0.10.4 --nics OnPrem-VMNIC

**On-Prem-VNG:**

**Note:** The VPN GW will take 20+ minutes to deploy

    az network public-ip create --name Azure-VNGpubip --resource-group Route-Server --allocation-method Dynamic --location northcentralus

    az network vnet-gateway create --name On-Prem-VNG --public-ip-address Azure-VNGpubip --resource-group Route-Server --vnet On-Prem-Vnet --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --asn 65001 --bgp-peering-address 10.0.0.4 --location northcentralus


- **Create the Azure HUB Network:**

**HUB-SCUS Vnet:**

    az network vnet create --resource-group Route-Server --name HUB-SCUS --location southcentralus --address-prefixes 10.1.0.0/16 --subnet-name Subnet-1 --subnet-prefix 10.1.10.0/24 
    az network vnet subnet create --address-prefix 10.1.0.0/24 --name External --resource-group Route-Server --vnet-name HUB-SCUS
    az network vnet subnet create --address-prefix 10.1.1.0/24 --name Internal --resource-group Route-Server --vnet-name HUB-SCUS
    az network vnet subnet create --address-prefix 10.1.2.0/27 --name RouteServerSubnet --resource-group Route-Server --vnet-name HUB-SCUS
    az network vnet subnet create --address-prefix 10.1.5.0/27 --name GatewaySubnet --resource-group Route-Server --vnet-name HUB-SCUS

**HUB-VM:**

    az network public-ip create --name HUB-VMPIP --resource-group Route-Server --location southcentralus --allocation-method Dynamic
    az network nic create --resource-group Route-Server -n HUB-VMNIC --location southcentralus --subnet Subnet-1 --private-ip-address 10.1.10.4 --vnet-name HUB-SCUS --public-ip-address  HUB-VMPIP
    az vm create -n HUB-VM -g Route-Server --image UbuntuLTS --admin-username azureuser --admin-password Routeserver123 --size Standard_B1ls --location southcentralus --private-ip-address 10.1.10.4 --nics HUB-VMNIC

**CSR NVA:**

Here we are going to use Cisco CSR1000v as NVA. This NVA will have two NICs, one external (CSROutsideInterface) and one Internal (CSRInsideInterface). Before deploying the ***CSR***, you need to accept license agreement:

    az vm image terms accept --urn cisco:cisco-csr-1000v:17_2_1-byol:latest 

    az network public-ip create --name CSRPublicIP --resource-group Route-Server --idle-timeout 30 --allocation-method Static --location southcentralus
    az network nic create --name CSROutsideInterface -g Route-Server --subnet External --vnet HUB-SCUS --public-ip-address CSRPublicIP --ip-forwarding true --private-ip-address 10.1.0.4 --location southcentralus
    az network nic create --name CSRInsideInterface -g Route-Server --subnet Internal --vnet HUB-SCUS --ip-forwarding true --private-ip-address 10.1.1.4 --location southcentralus
    az vm create --resource-group Route-Server --location southcentralus --name CSR --size Standard_DS3_v2 --nics CSROutsideInterface CSRInsideInterface --image cisco:cisco-csr-1000v:17_2_1-byol:17.2.120200508 --admin-username azureuser --admin-password Routeserver123


- **Build IPsec tunnel between ***CSR*** NVA (respresent Azure side) and ***On-Prem-VNG*** gateway (represent On-Premises side):**

**Note:** for simplicity, we will build one tunnel between ***CSR*** and ***On-Prem-VNG***

• After the ***On-Prem-VNG*** gateway and ***CSR*** have been created, document the public IP address for both, we will use them to build the Ipsec tunnel:

    az network public-ip show -g  Route-Server -n Azure-VNGpubip --query "{address: ipAddress}"
    az network public-ip show -g Route-Server -n CSRPublicIP --query "{address: ipAddress}"

• In below configuration, replace ***CSRPublicIP*** and ***Azure-VNGpubip*** with public IP addresses obtained from above.
	
• SSH to the CSR:
	
Go to command prompt and type:

    ssh azureuser@CSRPublicIP

• After providing the password `Routeserver123` and successfully login, you will see that you in enable mode (***#***) as shown below:

    CSR#

we will need to get into the configuration mode to configure Ipsec and BGP, so type (**conf t**):

    CSR#conf t

Now you in configuration mode:

    CSR(config)#

Paste in below configuration one block at a time, make sure to replace ***CSRPublicIP*** and ***Azure-VNGpubip*** with ips you got from earlier:
```
 crypto ikev2 proposal Azure-Ikev2-Proposal
 encryption aes-cbc-256
 integrity sha1 sha256
 group 2
!
crypto ikev2 policy Azure-Ikev2-Policy
 match address local 10.1.0.4 
 proposal Azure-Ikev2-Proposal
!
crypto ikev2 keyring to-onprem-keyring
 peer Azure-VNGpubip
  address Azure-VNGpubip
  pre-shared-key Routeserver
!
crypto ikev2 profile Azure-Ikev2-Profile
 match address local 10.1.0.4 
 match identity remote address Azure-VNGpubip 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local to-onprem-keyring
 lifetime 28800
 dpd 10 5 on-demand
!
crypto ipsec transform-set to-Azure-TransformSet esp-gcm 256
 mode tunnel
!
crypto ipsec profile to-Azure-IPsecProfile
 set transform-set to-Azure-TransformSet
 set ikev2-profile Azure-Ikev2-Profile
!
interface Loopback11
 ip address 192.168.1.1 255.255.255.255
!
interface Tunnel11
 ip address 192.168.2.1 255.255.255.255
 ip tcp adjust-mss 1350
 tunnel source 10.1.0.4
 tunnel mode ipsec ipv4
 tunnel destination Azure-VNGpubip
 tunnel protection ipsec profile to-Azure-IPsecProfile
!
router bgp 65002
 bgp router-id 192.168.1.1
 bgp log-neighbor-changes
 neighbor 10.0.0.4 remote-as 65001
 neighbor 10.0.0.4 ebgp-multihop 255
 neighbor 10.0.0.4 update-source Loopback11
 !
 address-family ipv4
  network 10.1.10.0 mask 255.255.255.0
  neighbor 10.0.0.4 activate
 exit-address-family
!
!Static route to On-Prem-VNG BGP ip pointing to Tunnel11, so that it would be reachable
ip route 10.0.0.4 255.255.255.255 Tunnel11
!Static route for Subnet-1 pointing to CSR default gateway of internal subnet, this is added in order to be able to advertise this route using BGP
ip route 10.1.10.0 255.255.255.0 10.1.1.1

```
Type `exit` multiple times until the prompt shows `csr#` or simply hit `CTRL Z` and save the configuration with `wr` as shown below:

    CSR# wr

• Now we need to create the tunnel (connection) from the on-premises side, and for that we will create the Local Network Gateway (LNG) and then the connection:

  **Create LNG:**

   az network local-gateway create --gateway-ip-address CSRPublicIP --name CSR --resource-group Route-Server --asn 65002 --bgp-peering-address 192.168.1.1 --local-address-prefixes 192.168.1.1/32 192.168.2.1/32

  **Create the connection:**

    az network vpn-connection create --name To-CSR --resource-group Route-Server --vnet-gateway1 On-Prem-VNG -l northcentralus --shared-key Routeserver --local-gateway2 CSR  --enable-bgp
    
    

## Task2: Verify Connectivity

**1.** Check the connection/tunnel status if it is connected: you can check it from the portal by navigating to Virtual Network Gateways -> On-Prem-VNG-> Settings -> Connections, it should show Connected as below:

![image](https://user-images.githubusercontent.com/78562461/139628348-ccad0fa6-0461-4dbd-a2c9-711968c6a564.png)


Or by using Azure Cli:

```
az network vpn-connection show --name To-CSR --resource-group Route-Server --query "{status: connectionStatus}"
{
  "status": "Connected"
}
```

**Note** that it can take sometimes to get updated from **Uknown** or **Not connected** to **Connected**.
	
You may also verify from the ***CSR*** if tunnel is up by checking on Tunnel11 if it shows interface and line protocol are `up`:

```
CSR#show interface tunnel11
Tunnel11 is up, line protocol is up
```

**2.** Check on BGP peers if they are connected and what routes have been exchanged:
	
   :point_right: **From CSR:** 
  
   - Check on BGP peers:
 
         Show ip bgp summary
	
      We see the BGP is up with peer 10.0.0.4 (***On-Prem-VNG***):

   ![image](https://user-images.githubusercontent.com/78562461/139735275-bce47ee1-8896-46c2-967a-64038a14b98b.png)


   - Check on routes exchanged through BGP
   
         show ip bgp
	
     it shows ***CSR*** is learning 10.0.0.0/16 which is ***On-Prem-Vnet*** prefix (AS path 65001), and it is advertising 10.1.10.0/24 (***Subnet-1*** prefix in ***HUB-SCUS*** Vnet).

   ![image](https://user-images.githubusercontent.com/78562461/139737638-2e3e4787-82eb-4bd3-bd4e-e926706d5001.png)
     
   :point_right: **From On-Prem-VNG**:

   Check on learned routes by either using the Azure Cli:
      
	        az network vnet-gateway list-learned-routes -g Route-Server -n On-Prem-VNG
	      
   or from Portal by navigating to Virtual Network Gateways -> On-Prem-VNG -> Monitoring -> BGP Peers:

   - 10.1.10.0/24 is ***Subnet-1*** prefix in ***HUB-SCUS*** vnet advertised by the ***CSR*** and so the AS Path shows 65002.
   
   - Routes shows local address same as source peer (10.0.0.4) and **origin** as **Network** are routes learned by the gateway instance locally, either by being the Vnet prefix          like 10.0.0.0/16, or by being static routes add to the LNG as for 192.168.1.1/32 and 192.168.2.1/32.


     ![image](https://user-images.githubusercontent.com/78562461/139993529-bf95a548-7a6c-4ddb-94cc-143e1382a32a.png)

**3.** Ping the ***On-Prem-VM*** (10.0.10.4) from ***CSR***, does it work?
	
**4.** SSH to the ***HUB-VM*** using its public ip, you can find it from the portal by navigating to Virtual Machines -> HUB-VM -> Overview -> public ip address, or using Cli `az network public-ip show -g Route-Server -n HUB-VMPIP --query "{address: ipAddress}` and ping ***On-Prem-VM*** (10.0.10.4), does it work?


**5.** SSH to ***On-Prem-VM***, get its public ip from portal same way as for ***HUB-VM***, or use Azure Cli `az network public-ip show -g Route-Server -n On-PremVMIP --query "{address: ipAddress}`, ping ***HUB-VM*** (10.1.10.4) does it work?



## Task3: Inspect the routing table to answer questions 3-5

👉 **For  3**, check on route table on the ***CSR***, do `show ip route 10.0.10.4` in enable mode as shown below:

![image](https://user-images.githubusercontent.com/78562461/139948143-dfda1cb5-b712-43ae-87a7-d383e7cfce2e.png)


We see that the ***CSR*** is learning the route through BGP over the tunnel from peer ip 10.0.0.4, which is the BGP peer ip of ***On-Prem-VNG***, and so when `Ping 10.0.10.4` it works:

```
CSR#ping 10.0.10.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.10.4, timeout is 2 seconds:
!!!!!
```

👉 **For  4**, check on route table of ***HUB-VM*** using `az network nic show-effective-route-table -g Route-Server -n HUB-VMNIC --output table`, we see that ***HUB-VM*** has no route to 10.0.10.0/24 and so when ping 10.0.10.4 it would fail:

```
$ az network nic show-effective-route-table -g Route-Server -n HUB-VMNIC --output table
Source                 State    Address Prefix    Next Hop Type          Next Hop IP
---------------------  -------  ----------------  ---------------------  -------------
Default                Active   10.1.0.0/16       VnetLocal
Default                Active   0.0.0.0/0         Internet


azureuser@HUB-VM:~$ ping 10.0.10.4
PING 10.0.10.4 (10.0.10.4) 56(84) bytes of data.

--- 10.0.10.4 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2036ms![image](https://user-images.githubusercontent.com/78562461/139951805-c2afc4db-164e-486e-87b6-d2afb3705776.png)

```

👉 **For  5,** Check route table of ***On-Prem-VM***  `az network nic show-effective-route-table -g Route-Server -n OnPrem-VMNIC  --output table`, we see ***On-Prem-VM*** is learning the route to the ***HUB-VM*** (as it is advertised through BGP from ***CSR*** over the tunnel), however, the ping will still fail as the ***HUB-VM*** doesn't have route back to 10.0.10.4 as shown above.


```	
$ az network nic show-effective-route-table -g Route-Server -n OnPrem-VMNIC --output table
Source                 State    Address Prefix    Next Hop Type          Next Hop IP
---------------------  -------  ----------------  ---------------------  -------------
Default                Active   10.0.0.0/16       VnetLocal
VirtualNetworkGateway  Active   192.168.1.1/32    VirtualNetworkGateway  20.X.X.X
VirtualNetworkGateway  Active   192.168.2.1/32    VirtualNetworkGateway  20.X.X.X
VirtualNetworkGateway  Active   10.1.10.0/24      VirtualNetworkGateway  20.X.X.X
Default                Active   0.0.0.0/0         Internet
```

💡 Traditionally, to have ***HUB-VM*** able to ping ***On-Prem-VM*** we need to create a UDR and associate it to ***subnet-1*** (where ***HUB-VM*** reside) to direct traffic destined to 10.0.10.0/24 (or in general to remote Vnet 10.0.0.0/16) to the ***CSR*** internal interface 10.1.1.4. But with ARS there is no need for UDR, ARS will inject the route it learns from ***CSR*** (NVA) automatically. We will see this next.



 
## Task4: Deploy ARS and check its effect on the route tables:

→ **Deploy ARS:**
	
	az network public-ip create --name RouteServerIP --resource-group Route-Server --version IPv4 --sku Standard --location southcentralus
	
	subnet_id=$(az network vnet subnet show --name RouteServerSubnet --resource-group Route-Server --vnet-name HUB-SCUS --query id -o tsv) 
	
	az network routeserver create --name RouteServer --resource-group Route-Server --hosted-subnet $subnet_id --public-ip-address RouteServerIP --location southcentralus
	

→ **Create the BGP peering between ARS peer ips and CSR internal interface 10.1.1.4:**
	
	az network routeserver peering create --name CSR --peer-ip 10.1.1.4 --peer-asn 65002 --routeserver RouteServer --resource-group Route-Server
	
Document the BGP peer ips of the ARS, we will need them to complete the configuration on the ***CSR*** to establish BGP session with ARS.

:exclamation: Note: as of now ARS ASN number is not configurable and it is hard coded to 65515

```
$ az network routeserver show --name RouteServer --resource-group route-server
	.
	.
	.
	.
  "virtualRouterAsn": 65515,
  "virtualRouterIps": [
    "10.1.2.5", <===
    "10.1.2.4" <====
```

→ **SSH to ***CSR*** to update the BGP configuration, once login type `conf t` to get into configuration mode**:

	CSR#conf t
 
 Now copy and paste the following one block at a time:
 

```
         router bgp 65002
	 neighbor 10.1.2.4 remote-as 65515
	 neighbor 10.1.2.4 ebgp-multihop 255
	 neighbor 10.1.2.5 remote-as 65515
	 neighbor 10.1.2.5 ebgp-multihop 255
	 !
	 address-family ipv4
	  neighbor 10.1.2.4 activate
	  neighbor 10.1.2.5 activate
	 exit-address-family
	!
	! add static route to ARS subnet that point to the default gateway of the CSR Internal subnet to avoid recursive routing failure for ARS BGP endpoints learned via BGP
	ip route 10.1.2.0 255.255.255.0 10.1.1.1
```

Type `exit` multiple times or simply press `Ctrl Z` to get into enable mode `#`, then type `wr` to save above config:

    CSR# wr


→ **Verify if BGP has established between ARS and ***CSR***:**
	
	
On the ***CSR*** do `show ip bgp summary`: we see BGP is up with 10.1.2.4 and 10.1.2.5 which are the ARS BGP endpoints:


![image](https://user-images.githubusercontent.com/78562461/139961525-8354434a-2177-462e-865b-49feff50061c.png)


→ **Check now if we can ping from ***HUB-VM*** to ***On-prem-VM*** and vice versa, does it work?**

🙂 **Let inspect routing table on ***RouteServer***, ***CSR***, ***On-prem-VNG*** gateway, ***On-Prem-VM***, and ***HUB-VM*** after deploying the ARS to answer above question:**
	 
**1. RouteServer Routes:**

   :exclamation: Note: we can check on learned/advertised routes per peer at a time, which is here we have one peer (***CSR***):
   
   :point_right: Learned Routes from ***CSR***:

      az network routeserver peering list-learned-routes --name CSR --routeserver RouteServer --resource-group Route-Server
	
**·** We see all routes have next hop as 10.1.1.4 which is the ***CSR*** internal (LAN) interface that the ARS ***RouteServer*** is peering with. Pay attention to the asPath, the one with 65002-65001 shows this route has been advertised from (***On-Prem-VNG***) which is here 10.0.0.0/16, while route 10.1.10.0/24 with asPath 65002 is a route been advertised by the ***CSR*** itself .

❗ ARS has two instances for high availabilty, IN_0 (10.1.2.4) and IN_1 (10.1.2.5), and both will have same learned/advertised routes.

![image](https://user-images.githubusercontent.com/78562461/140400321-3c67f0e0-17a3-4ce3-979e-e600260371b5.png)

   :point_right: Advertised Routes to ***CSR***:
   
    az network routeserver peering list-advertised-routes --name CSR --routeserver RouteServer --resource-group Route-Server
    
   ARS advertised the ***HUB-SCUS*** prefix (10.1.0.0/16) to the ***CSR***:

```
$ az network routeserver peering list-advertised-routes --name CSR --routeserver RouteServer --resource-group Route-Server
{
  "RouteServiceRole_IN_0": [
    {
      "asPath": "65515",
      "localAddress": "10.1.2.4",
      "network": "10.1.0.0/16", <===== HUB-SCUS Vnet Prefix
      "nextHop": "10.1.2.4",
      "origin": "Igp",
      "weight": 0
    },
   
  ],
  "RouteServiceRole_IN_1": [
    {
      "asPath": "65515",
      "localAddress": "10.1.2.5",
      "network": "10.1.0.0/16",
      "nextHop": "10.1.2.5",
      "origin": "Igp",
      "weight": 0
    },
  ```


**2. CSR (NVA) Routes:**
	
:point_right: Check on BGP routes by typing `show ip bgp` in enable mode:
```
CSR#show ip bgp
BGP table version is 5, local router ID is 192.168.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.0.0.0/16      10.0.0.4                               0 65001 i
 *    10.1.0.0/16      10.1.2.5                               0 65515 i
 *>                    10.1.2.4                               0 65515 i
 *>   10.1.10.0/24     10.1.1.1                 0         32768 i
 ```
	
- Route with Next Hop 10.1.2.4 and 10.1.2.5 and ASN 65515 is a route learned from ARS, and here it shows the ***CSR*** learned the ***HUB-SCUS*** Vnet prefix (10.1.0.0/16)
  from ARS, and it will send it then through eBGP over IPsec to ***On-prem-VNG***, this way ***On-Prem-Vnet*** will learn the ***HUB-SCUS*** prefix. Note that the route
  10.1.0.0/16 is learned from the two Route Server instances 10.1.2.4 and 10.1.2.5, but the path through 10.1.2.4 has been chosen as best (**>**) based on [BGP Best Path Selection Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html).

	
- Route with Next Hop 10.0.0.4 and ASN 65001 is a route received from ***On-Prem-VNG***, which is here 10.0.0.0/16, ***CSR*** will send it to ARS (as we saw above in ARS learned routes) through eBGP, and ARS in turn will advertise it to the ***HUB-SCUS*** Vnet (will see that later).
	
- Route with Next Hop 10.1.1.1 and no ASN but with weight 32768 is a route advertised by the ***CSR*** itself, here it is the route 10.1.10.0/24 which refer to ***HUB-VM*** subnet (***Subnet-1***).


:point_right: Check on effective routes on the ***CSR*** NICs, either by using the portal or by using the following Cli commands:
  
   ```
 az network nic show-effective-route-table -g Route-Server -n CSROutsideInterface --output table
 az network nic show-effective-route-table -g Route-Server -n CSRInsideInterface --output table
   ```
   
* Both NICs return the below table. Note that 10.0.0.0/16 (***On-Prem-Vnet***) got injected by ARS in the ***CSR*** NICs' route table, with Next Hop IP as the ***CSR*** LAN Interface and Next Hop Type as Virtual Network Gateway, so traffic will be directed to the ***CSR*** directly. This shows that **ARS is not in the data path**, it only exchange BGP routes with NVA and program the routes it learns in the NICs' route table.

:exclamation: We don't see 10.1.10.0/24 route programmed in the NICs' route table, this route is ***Subnet-1*** prefix that belong to the Vnet (***HUB-SCUS***), even though it is learned by the ARS from the ***CSR*** as shown in ARS learned routes above, why?

:bulb: Because ARS will not program any route equal to the Vnet/Subnets prefix into the Vnet. In other words, ARS will not program routes it knows through system routes and so we cannot override the Vnet system routes using ARS.

```

$ az network nic show-effective-route-table -g Route-Server -n CSROutsideInterface --output table
Source                 State    Address Prefix    Next Hop Type          Next Hop IP
---------------------  -------  ----------------  ---------------------  -------------
Default                Active   10.1.0.0/16       VnetLocal
VirtualNetworkGateway  Active   10.0.0.0/16       VirtualNetworkGateway  10.1.1.4
Default                Active   0.0.0.0/0         Internet
Default                Active   10.0.0.0/8        None
Default                Active   100.64.0.0/10     None
.
.
.
```

**3. Effective Route on HUB-VM:** 

 Use Azure Cli:
 
     az network nic show-effective-route-table -g Route-Server -n HUB-VMNIC --output table
	
We see ***On-Prem-Vnet*** prefix 10.0.0.0/16 is programed by ARS in the NIC route table with Next Hop IP as the ip of the LAN interface of the ***CSR***:

![image](https://user-images.githubusercontent.com/78562461/140001830-19a3fba2-aabf-4806-979b-70a5996bcd49.png)

**4. Learned routes by the On-Prem-VNG gateway:**
	
- Check on learned routes from the portal by navigating to Virtual Network Gateways -> On-Prem-VNG -> Monitoring -> BGP Peers or use Azure Cli `az network vnet-gateway list-learned-routes -g Route-Server -n On-Prem-VNG`.
	
- Next hop 192.168.1.1 is the Loopback 11 that has been used as update source for BGP in the ***CSR***, this address has been added to the LNG of the ***On-Prem-VNG*** so it knows how to get to this BGP peer ip, and also added 192.168.2.1 (Tunnel11 ip in the ***CSR***) to the LNG, that is why **Origin** to these two addresses shows as **Network**. 
	
- Route 10.1.10.0/24 is the ***subnet-1*** prefix in the ***HUB-SCUS*** Vnet that is advertised by Network command in the ***CSR***, and so we see the AS path has 65002 which is ***CSR*** ASN. 
	
- Route 10.1.0.0/16 is the ***HUB-SCUS*** Vnet prefix, learned through the ***CSR***(192.168.1.1), it is advertised originally from the ARS as illustrated by the As path 65002-65515, in which 65515 is the ARS ASN.
	
![image](https://user-images.githubusercontent.com/78562461/140002829-5b74b986-8ea9-41ff-82d6-dffcb796705d.png)


	
**5. Effective Route on On-Prem-VM:**
	
- Use Azure Cli  `az network nic show-effective-route-table -g Route-Server -n OnPrem-VMNIC --output table`
	
- All routes learned by the ***On-Prem-VNG*** gateway are injected in the NIC with the Next Hop as the public ip of the ***On-prem-VNG*** gateway (20.X.X.X in this example).

```
$ az network nic show-effective-route-table -g Route-Server -n OnPrem-VMNIC --output table
Source                 State    Address Prefix    Next Hop Type          Next Hop IP
---------------------  -------  ----------------  ---------------------  -------------
Default                Active   10.0.0.0/16       VnetLocal
VirtualNetworkGateway  Active   10.1.0.0/16       VirtualNetworkGateway  20.X.X.X
VirtualNetworkGateway  Active   192.168.1.1/32    VirtualNetworkGateway  20.X.X.X
VirtualNetworkGateway  Active   192.168.2.1/32    VirtualNetworkGateway  20.X.X.X
VirtualNetworkGateway  Active   10.1.10.0/24      VirtualNetworkGateway  20.X.X.X
Default                Active   0.0.0.0/0         Internet

```

👉 The complete route exchange through BGP between on-premises and Azure is shown below:

![image](https://user-images.githubusercontent.com/78562461/140006065-29623306-8c94-4483-90c0-38cb5e6e5f0a.png)

🙂 From above we can see that ***HUB-VM*** has route now to ***On-Prem-VM*** after introducing ARS with no UDR needed, so ping will work:


```
azureuser@HUB-VM:~$ ping 10.0.10.4
PING 10.0.10.4 (10.0.10.4) 56(84) bytes of data.
64 bytes from 10.0.10.4: icmp_seq=1 ttl=63 time=35.8 ms
64 bytes from 10.0.10.4: icmp_seq=2 ttl=63 time=36.8 ms
64 bytes from 10.0.10.4: icmp_seq=3 ttl=63 time=36.7 ms
64 bytes from 10.0.10.4: icmp_seq=4 ttl=63 time=36.9 ms
^C
--- 10.0.10.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 35.894/36.614/36.929/0.418 ms
```

## Scenario 2: Route exchange between Virtual Network Gateway and CSR NVA 

In this scenario we will connect another on-premises network in Central US to Vnet ***HUB-SCUS*** using Azure Virtual Network Gateway. You will explore how ARS can help in exchanging routes between the Virtual Network Gateway and the CSR, and how will that affect the overall connectivity between endpoints in this scenario.

	
In this scenario we will add the following component:

1- ***HUB-VNG*** represent virtual network gateway in the ***HUB-SCUS*** Vnet in active-active mode.

2- New Vnet called ***On-Prem1-Vnet*** represent on-premises network in Central US, this Vnet will have ***On-Prem1-VNG*** which represent on-premises gateway, and will also have testing VM ***On-Prem1-VM***.

3- IPsec site to site single tunnel between the two virtual network gateways (***Hub-VNG*** and ***On-Prem1-VNG***) using BGP as the dynamic routing protocol.

Once above get deployed the complete diagram will look as below:

![image](https://user-images.githubusercontent.com/78562461/141018182-f6b4e14e-cc5d-42d7-aef6-dd9ccc47ec51.png)


## Task1: Deploy the ***HUB-VNG*** virtual network gateway in the ***HUB-SCUS***

	
- Create BGP enabled gateway in active-active mode, it takes 20+ minutes to get deployed:
	
:exclamation:To allow route transit between ARS and virtual network gateway, the latter need to be in [active-active](https://docs.microsoft.com/en-us/azure/route-server/expressroute-vpn-support#how-does-it-work) mode.
	
```	
az network public-ip create --name HUB-VNG-PIP1 --resource-group Route-Server --allocation-method Dynamic --location southcentralus
az network public-ip create --name HUB-VNG-PIP2 --resource-group Route-Server --allocation-method Dynamic --location southcentralus 
az network vnet-gateway create --name HUB-VNG --public-ip-address HUB-VNG-PIP1 HUB-VNG-PIP2 --resource-group Route-Server --vnet HUB-SCUS --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --asn 65004  --location southcentralus
```

## Task2: Build the on-premises network (***On-Prem1-Vnet***) and its components
	
	
- ***On-Prem1-Vnet***:
```	
az network vnet create --resource-group Route-Server --name On-Prem1-Vnet --location centralus --address-prefixes 10.2.0.0/16 --subnet-name Subnet-1 --subnet-prefix 10.2.10.0/24 
az network vnet subnet create --address-prefix 10.2.2.0/27 --name GatewaySubnet --resource-group Route-Server --vnet-name On-Prem1-Vnet
```

- ***On-Prem1-VM***:
```	
az network public-ip create --name On-Prem1-VMPIP --resource-group Route-Server --location centralus --allocation-method Dynamic
az network nic create --resource-group Route-Server -n On-Prem1-VMNIC --location centralus --subnet Subnet-1 --private-ip-address 10.2.10.4 --vnet-name On-Prem1-Vnet --public-ip-address On-Prem1-VMPIP
az vm create -n On-Prem1-VM -g Route-Server --image UbuntuLTS --admin-username azureuser --admin-password Routeserver123 --size Standard_B1ls --location centralus --nics On-Prem1-VMNIC
```

- ***On-Prem1-VNG***. It takes 20+ minutes to get deployed:

```
az network public-ip create --name OnPrem1VNG-PIP1 --resource-group Route-Server --allocation-method Dynamic --location centralus
az network vnet-gateway create --name On-Prem1-VNG --public-ip-address OnPrem1VNG-PIP1  --resource-group Route-Server --vnet On-Prem1-Vnet --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --asn 65003 --bgp-peering-address 10.2.2.30 --location centralus
```


## Task3: Establish the BGP connection/tunnel between Hub-VNG gateway and On-Prem1-VNG gateway

- Document the ***HUB-VNG-PIP1*** and do the same for the ***OnPrem1VNG-PIP1***, we will need them to configure Local Network Gateway for each side:
	
	   az network public-ip show -g  Route-Server -n  OnPrem1VNG-PIP1 --query "{address: ipAddress}"
	   az network public-ip show -g  Route-Server -n  HUB-VNG-PIP1 --query "{address: ipAddress}"
	
- Document the Default Azure BGP peer ip address of the ***HUB-VNG*** gateway as will also be needed to create the HUB local network gateway (***HUB-LNG***), usually it would be 10.1.5.4 but need to check as it can also be any other ip in the Gateway Subnet, note that this Cli command will also list the corresponding public ip "***OnPrem1VNG-PIP1***" that you got from above command:
	
	  az network vnet-gateway show -g Route-Server -n HUB-VNG
	  
```
	  Output example:
	
	 {
	  "active": true,
	  "bgpSettings": {
	    "asn": 65004,
	    "bgpPeeringAddress": "10.1.5.4,10.1.5.5",
	    "bgpPeeringAddresses": [
	      {
	        "customBgpIpAddresses": [],
	        "defaultBgpIpAddresses": [ <======
	          "10.1.5.4" <===== BGP peer ip 
	        ],
	        "ipconfigurationId": "/subscriptions/c04dcf3e-863e-4927-8497-511ab285fa8f/resourceGroups/Route-Server/providers/Microsoft.Network/virtualNetworkGateways/HUB-VNG/ipConfigurations/vnetGatewayConfig0",
	        "tunnelIpAddresses": [
	          "40.124.140.162" <======HUB-VNG-PIP1 in this example
	        ]
	      },
	      {
```
       
You can check on same from portal, navigate to Virtual Network Gateway -> HUB-VNG -> Settings -> Configuration, check on Default Azure BGP peer ip address, and you can also find ***HUB-VNG-PIP1*** you got from earlier here:

![image](https://user-images.githubusercontent.com/78562461/140182294-9ffdb5b4-3167-4b25-af6a-08b05b956d83.png)


- Create the local network gateway ***HUB-LNG*** that represent the ***HUB-VNG*** gateway, replace ***HUB-VNG-PIP1*** with the ip you documented, also replace `10.1.5.X` with the Default Azure BGP peer ip address you got from above:
	
      az network local-gateway create --gateway-ip-address HUB-VNG-PIP1 --name HUB-LNG --resource-group Route-Server --asn 65004 --bgp-peering-address 10.1.5.X
	
- Create the local network gateway ***On-prem1-LNG*** that represent ***On-Prem1-VNG*** gateway, replace ***OnPrem1VNG-PIP1***  with the one you documented from earlier (Note: Default BGP ip address has been already hard coded when the gateway created to 10.2.2.30 so no need to change it):
	
      az network local-gateway create --gateway-ip-address OnPrem1VNG-PIP1  --name On-Prem1-LNG --resource-group Route-Server --asn 65003 --bgp-peering-address 10.2.2.30
	
- Create the connection/tunnel between the gateways, note we will have only one tunnel between ***HUB-VNG*** and ***On-Prem1-VNG***:
	
      az network vpn-connection create --name To-HUB-SCUS --resource-group Route-Server --vnet-gateway1 On-Prem1-VNG -l centralus --shared-key Routeserver --local-gateway2 HUB-LNG --enable-bgp
      az network vpn-connection create --name To-On-Prem1 --resource-group Route-Server --vnet-gateway1 HUB-VNG -l southcentralus --shared-key Routeserver --local-gateway2  On-Prem1-LNG --enable-bgp
	
- Check on connection status:

You can check one connection from either side to get the status of the tunnel. Note that it take some time to get updated from Unknown to Connected
	
    az network vpn-connection show --name To-HUB-SCUS --resource-group Route-Server --query "{status: connectionStatus}"

## Task4: Configure route exchange

- To exchange routes between VPN gateway (***HUB-VNG***) and the ARS, enable Branch-to-Branch feature:
	
	  az network routeserver update --name RouteServer --resource-group Route-Server --allow-b2b-traffic true
	
## Task5: Verify connectivity

→ Ping from ***On-Prem-VM*** (10.0.10.4) to ***On-Prem1-VM*** (10.2.10.4), does it work? Why?

✋ To answer above question let explore the route tables along the path starting from source VM to destination VM to check how the routes have been exchanged: 

	
1- **Check on routing from source VM (***On-Prem-VM***):**
	
Navigate to Network interfaces -> OnPrem-VMNIC -> Help -> Effective routes
	
or use Cli:
	
az network nic show-effective-route-table -g Route-Server -n onprem-VMNIC --output table
	
- 10.1.0.0/16 is ***HUB-SCUS*** Vnet prefix.
- 10.2.0.0/16 is ***On-Prem1-Vnet*** prefix where the destination ***On-Prem1-VM*** is located, this shows that source VM has learned the route to destination VM.
- 192.168.1.1 is ***CSR*** BGP peer ip and 192.168.2.1 is ***CSR*** Tunnel11 ip.
- 10.1.10.0/24 is ***Subnet-1*** in ***HUB-SCUS*** Vnet (this prefix is advertised by the ***CSR*** using Network command).

All those routes are showing **Next Hop Type** as **Virtual Network Gateway** which refer to the ***On-Prem-VNG*** gateway. Next we will check how this gateway learned those routes.
	
```
$ az network nic show-effective-route-table -g Route-Server -n onprem-VMNIC --output table
Source                 State    Address Prefix    Next Hop Type          Next Hop IP
---------------------  -------  ----------------  ---------------------  --------------
Default                Active   10.0.0.0/16       VnetLocal
VirtualNetworkGateway  Active   192.168.2.1/32    VirtualNetworkGateway  52.X.X.X
VirtualNetworkGateway  Active   10.2.0.0/16       VirtualNetworkGateway  52.X.X.X
VirtualNetworkGateway  Active   10.1.10.0/24      VirtualNetworkGateway  52.X.X.X
VirtualNetworkGateway  Active   10.1.0.0/16       VirtualNetworkGateway  52.X.X.X
VirtualNetworkGateway  Active   192.168.1.1/32    VirtualNetworkGateway  52.X.X.X
Default                Active   0.0.0.0/0         Internet
```

2. **Check ***On-Prem-VNG*** gateway:** 
	
Navigate to Virtual Network Gateways -> On-Prem-VNG -> Monitoring -> BGP Peers:
	
👉 **BGP Peers:**
	
- The local gateway BGP ip 10.0.0.4 is peering successfully with ***CSR*** BGP peer ip 192.168.1.1. Let check next on routes learned due to this peering. 
	
	![image](https://user-images.githubusercontent.com/78562461/140189765-fb819b17-4f70-4ff9-a1bf-e98026996a19.png)

	
	
👉 **Learned Routes:** 
	
- 10.2.0.0/16 is the Vnet prefix of the destination VM ***On-Prem1-VM***. Check the AS Path to know how the gateway learned the route to the destination: ASN 65003 refer to the destination gateway (***On-Prem1-VNG***) which originally advertised this route to the ***HUB-VNG*** gateway that has the ASN 65004 through eBGP peering, this route then advertised to the Route Server as **Branch-to-Branch** feature is enabled, so we see the ASN 65515 in the AS Path, then Route Server advertised it to the ***CSR*** which has the ASN 65002, and ***CSR*** advertise it down to the ***On-Prem-VNG***.
	
- 10.1.0.0/16 is ***HUB-SCUS*** Vnet prefix that is learned by the gateway from ***CSR*** ASN 65002 through eBGP peering over IPsec, note that this prefix originally advertised by the Route Server through eBGP peering to the ***CSR***, and that why we see 65002-65515 in the AS path.
	
- 10.1.10.0/24 is ***Subnet-1*** prefix in the ***HUB-SCUS*** Vnet, the AS Path shows only 65002 which is the ASN of ***CSR*** as this route is advertised using Network command in the ***CSR*** BGP configuration (CSR BGP configuration is shown below).

![image](https://user-images.githubusercontent.com/78562461/140190954-78a8f084-9e5d-43e4-942c-eb8e9e229bc2.png)

```
CSR#show running-config | sec bgp
router bgp 65002
 bgp router-id 192.168.1.1
 bgp log-neighbor-changes
 neighbor 10.0.0.4 remote-as 65001
 neighbor 10.0.0.4 ebgp-multihop 255
 neighbor 10.0.0.4 update-source Loopback11
 neighbor 10.1.2.4 remote-as 65515
 neighbor 10.1.2.4 ebgp-multihop 255
 neighbor 10.1.2.5 remote-as 65515
 neighbor 10.1.2.5 ebgp-multihop 255
 !
 address-family ipv4
  network 10.1.10.0 mask 255.255.255.0 <=====
  neighbor 10.0.0.4 activate
  neighbor 10.1.2.4 activate
  neighbor 10.1.2.5 activate
 exit-address-family
 
```
3. **Check CSR:**
	
👉 **BGP Peers:** 

`show ip bgp summary` 
	
- 10.0.0.4 is the BGP peer ip for ***On-Prem-VNG***, 10.1.2.4 and 10.1.2.5 are BGP peer ips of the Route Server. We see BGP has been established with all of the peers

```
CSR#show ip bgp sum
CSR#show ip bgp summary
BGP router identifier 192.168.1.1, local AS number 65002
BGP table version is 6, main routing table version 6
5 network entries using 1240 bytes of memory
8 path entries using 1088 bytes of memory
4/4 BGP path/bestpath attribute entries using 1152 bytes of memory
3 BGP AS-PATH entries using 88 bytes of memory
1 BGP community entries using 24 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 3592 total bytes of memory
BGP activity 5/0 prefixes, 8/0 paths, scan interval 60 secs
5 networks peaked at 21:42:19 Nov 3 2021 UTC (00:05:34.882 ago)

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.0.4        4        65001      60      59        6    0    0 00:49:56        1
10.1.2.4        4        65515      60      60        6    0    0 00:50:01        2
10.1.2.5        4        65515      60      60        6    0    0 00:50:02        2
 ```
 	
👉 **BGP route table:**

`show ip bgp`
	
- 10.0.0.0/16 is learned from ***On-prem-VNG*** (ASN 65001).
	
- 10.2.0.0/16 is the destination prefix that is learned from the Route Server instances 10.1.2.4 and 10.1.2.5. Check the AS Path to track how the ***CSR*** learned this prefix: ASN 65003 refer to ***On-prem1-VNG*** that own this prefix 10.2.0.0/16, then 65004 is the ASN of ***HUB-VNG*** gateway which learned this prefix from ***On-Prem1-VNG***, Route Server with ASN 65515 then learned this prefix and advertised it to ***CSR*** which in turn advertised it down to ***On-Prem-VNG*** as we saw earlier.  
	
- 10.1.0.0/16 is the ***HUB-SCUS*** prefix where the ***CSR*** reside, this prefix learned from the Route Server instances, the ***CSR*** will advertise it then to the ***On-Prem-VNG*** gateway.

![image](https://user-images.githubusercontent.com/78562461/140201163-4eb56756-dcc9-4663-ae59-7bae588e1b44.png)



4. **Check on ARS (RouteServer) routes:**
	
👉 **Check on routes learned from CSR NVA:**

    az network routeserver peering list-learned-routes --name CSR --routeserver RouteServer --resource-group Route-Server
	
- Both ARS instances IN_0 (10.1.2.5) and IN_1 ( 10.1.2.4) learn the same routes from the ***CSR***. 
	
- 10.1.10.0/24 is ***Subnet-1*** prefix that the ***CSR*** (ASN 65002) advertised manually, 10.0.0.0/16 is ***On-Prem-Vnet*** prefix that is learned from ***On-Prem-VNG*** (ASN 65001) and the ***CSR*** (65002) advertised it then to ARS. 


![image](https://user-images.githubusercontent.com/78562461/140200447-531d93bd-8197-413a-820f-18e1578ea3e0.png)

👉 **Check on Routes advertised from ARS to the CSR:**

    az network routeserver peering list-advertised-routes --name CSR --routeserver RouteServer --resource-group Route-Server
	
- Although ARS will not inject the ***HUB-SCUS*** prefix 10.1.0.0/16 into the NICs effective routes, it will advertise this prefix to the ***CSR*** over eBGP, so ***CSR*** can advertise it down to ***On-Prem-VNG*** gateway.
	
- 10.2.0.0/16 is learned from ***HUB-VNG (65004)*** which has learned it from ***On-Prem1-VNG*** (65003).

![image](https://user-images.githubusercontent.com/78562461/140241249-f9037e9e-1847-4a4b-be9b-ea283981498f.png)

	
5. **Check on HUB-VNG:**
	
👉 **BGP Peers:** Navigate to Virtual Network Gateways -> HUB-VNG -> Monitoring -> BGP peers
	
- Peer addresses in yellow boxes with ASN 65515 are the Route Server BGP peer ips peering with the two ***HUB-VNG*** gateway instances (10.1.5.5 and 10.1.5.4), and all are successfully connected.
	
- Peer address 10.2.2.30 in red box with ASN 65003 is the BGP peer ip of the ***On-Prem1-VNG*** gateway. Note that we only see it in "connected" state with instance 10.1.5.4 as we have built only one tunnel between the two gateways and we used this ***HUB-VNG*** BGP ip 10.1.5.4 to peer with ***On-Prem1-VNG*** gateway.


![image](https://user-images.githubusercontent.com/78562461/140241905-f02546c1-46dd-41fb-b56e-6dc38e156da4.png)

👉 **Learned Routes:**
	
:exclamation: Output will show learned routes for the two gateway instances of the ***HUB-VNG*** gateway, however, as only one tunnel has been built with ***On-Prem1-VNG***, output below is filtered for the one instance used to build this tunnel, which is local address 10.1.5.4 of the ***HUB-VNG*** (Remember this local address BGP ip "might" be different in your lab).
	
- It can be seen that the ***HUB-VNG*** is learning the source prefix (10.0.0.0/16) through this AS Path: 65001 (***On-Prem-VNG***), 65002 (***CSR***), and 65515 (***Route Server***). It also learned the destination prefix (10.2.0.0/16) directly from ASN 65003 (***On-Prem1-VNG***).

![image](https://user-images.githubusercontent.com/78562461/140245775-a38fda90-bc11-447c-bd40-5ab0fb2d6a6c.png)


6. **Check On-Prem1-VNG:**
 
 Navigate to Virtual Network Gateways ->  On-Prem1-VNG -> Monitoring -> BGP Peers

👉 **BGP Peers:** 
	
- 10.1.5.4 is the BGP peer address of ***HUB-VNG*** gateway, 10.2.2.30 is the local BGP ip for the ***On-Prem1-VNG*** gateway. From below we see they are successfully connected.

![image](https://user-images.githubusercontent.com/78562461/140246508-75fcae89-2fc4-4ce4-ab97-a021fe1bb4f4.png)


👉 **Learned Routes:** 
	
- 10.0.0.0/16 is ***On-Prem-Vnet*** prefix which is where the source VM (***On-Prem-VM***) is located. Let us look at the AS Path which explain how this route has been learned: ASN 65001 refer to ***On-Prem-VNG*** gateway, it then advertised it through eBGP peering over IPsec to ***CSR*** NVA which has ASN 65002, and as the ***CSR*** has eBGP peering with ARS, it advertised this route to the ARS which has the ASN 65515. ARS has the **Branch-to-Branch** feature enabled, so it advertised this route to the ***HUB-VNG*** gateway which has the ASN 65004, and the ***HUB-VNG*** advertised it down to the ***On-Prem1-VNG*** gateway through the eBGP peering over IPsec.
	
💡 Without **Branch-to-Branch** feature enabled on ARS, 10.0.0.0/16 will not be advertised from ***CSR*** to the ***HUB-VNG***, and 10.2.0.0/16 will not be advertised from ***HUB-VNG*** gateway to the ***CSR***. This feature is also used to enable route transit between VPN gateway and express route gateway which was not possible before.
	
- 10.1.0.0/16 is learned through eBGP peering over IPsec from peer 10.1.5.4.

![image](https://user-images.githubusercontent.com/78562461/140253979-64f592e6-2b66-4001-8376-1c2091dee1e0.png)


7. **Lastly**, ***On-Prem1-VM***:

 Navigate to Network interfaces -> On-Prem1-VMNIC -> Help -> Effective routes
	
  or use Cli:
	
az network nic show-effective-route-table -g Route-Server -n on-prem1-VMNIC --output table
	
- We can see that the VM learned 10.0.0.0/16 where the source VM located, 10.1.5.4 is the BGP peer ip of the ***HUB-VNG*** gateway, 10.1.0.0/16 is the ***HUB-SCUS*** Vnet prefix, all those routes have been learned with Next Hop Type **virtual network gateway** which refer to ***On-Prem1-VNG*** gateway.

	
```
$ az network nic show-effective-route-table -g Route-Server -n on-prem1-VMNIC --output table
Source                 State    Address Prefix    Next Hop Type          Next Hop IP
---------------------  -------  ----------------  ---------------------  --------------
Default                Active   10.2.0.0/16       VnetLocal
VirtualNetworkGateway  Active   10.1.5.4/32       VirtualNetworkGateway  23.X.X.X
VirtualNetworkGateway  Active   10.1.0.0/16       VirtualNetworkGateway  23.X.X.X
VirtualNetworkGateway  Active   10.0.0.0/16       VirtualNetworkGateway  23.X.X.X
Default                Active   0.0.0.0/0         Internet
```


🙂 From above we see that source ***On-Prem-VM*** (10.0.10.4) knows the route to destination ***On-Prem1-VM*** (10.2.10.4) and vice versa with no UDR has been used due to using ARS, and we see ping work fine:

```
azureuser@On-Prem-VM:~$ ping 10.2.10.4
PING 10.2.10.4 (10.2.10.4) 56(84) bytes of data.
64 bytes from 10.2.10.4: icmp_seq=1 ttl=63 time=64.8 ms
64 bytes from 10.2.10.4: icmp_seq=2 ttl=63 time=63.6 ms
^C
--- 10.2.10.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 63.651/64.256/64.862/0.656 ms
```

![image](https://user-images.githubusercontent.com/78562461/140256474-18683fe7-3140-48a9-8cbd-81f1d47f7fb9.png)



## Scenario 3: Vnet peering and ARS

In this scenario we will explore how a peered Vnet can learn routes from ARS that is deployed in peered Vnet, what will the peered ***Spoke-VM*** learn and what the next hop will be when having Virtual Network Gateway combined with ARS in same remote Vnet.

  
For this scenario we will add the following component:

1. ***Spoke-Vnet*** that will peer with the ***HUB-SCUS*** Vnet where the ARS is deployed. 

2. ***Spoke-VM*** to check on routing and test connectivity.

After deploying above the network will look as below:

![image](https://user-images.githubusercontent.com/78562461/145701573-297fd361-0135-4b23-9add-91aa2900d57b.png)

	
## Task1: Create the Spoke-Vnet and the Spoke-VM

- Create the Spoke-Vnet:
```
 az network vnet create --resource-group Route-Server --name Spoke-Vnet --location westus --address-prefixes 10.4.0.0/16 --subnet-name Subnet-1 --subnet-prefix 10.4.10.0/24 
```

- Create the Spoke-VM
```
az network public-ip create --name Spoke-VMPIP --resource-group Route-Server --location westus --allocation-method Dynamic
az network nic create --resource-group Route-Server -n Spoke-VMNIC --location westus --subnet Subnet-1 --private-ip-address 10.4.10.4 --vnet-name Spoke-Vnet --public-ip-address Spoke-VMPIP
az vm create -n Spoke-VM -g Route-Server --image UbuntuLTS --admin-username azureuser --admin-password Routeserver123 --size Standard_B1ls --location westus --nics Spoke-VMNIC
```

## Task2: Create the peering

- You must first get the ID of each virtual network and store the ID in a variable:

```	
vNet1Id=$(az network vnet show --resource-group Route-Server --name Spoke-Vnet --query id --out tsv)
```
```
vNet2Id=$(az network vnet show --resource-group Route-Server --name HUB-SCUS --query id --out tsv)
```

- Create peering from Spoke-To-HUB: 
	
Note: We will enable `Use Remote Gateway`that allow the ***Spoke-Vnet*** to learn the routes from Virtual Network Gateway and/or Azure Route Server that are deployed in the remote Vnet

```
az network vnet peering create --name Spoke-To-HUB --resource-group Route-Server --vnet-name Spoke-Vnet --remote-vnet $vNet2Id --allow-vnet-access --use-remote-gateways
```

- Create peering from HUB-To-Spoke

```	
az network vnet peering create --name HUB-To-Spoke --resource-group Route-Server --vnet-name HUB-SCUS --remote-vnet $vNet1Id --allow-vnet-access --allow-gateway-transit
```

## Task3: Explore routing tables:

1. **Check on Spoke-VM:**
 
 Navigate to Network Interfaces -> Spoke-VMNIC -> Help -> Effective routes
	
- 10.2.0.0/16 is the ***On-Prem1-Vnet*** prefix that is learned by the ***HUB-VNG*** gateway from ***On-Prem1-VNG***, and as the ***HUB-VNG*** is active-active we see this route showing twice, one learned from gateway instance 10.1.5.5 and one from the other instance 10.1.5.4.
	
- 10.0.0.0/16 is the ***On-Prem-Vnet*** prefix that is learned from ***CSR*** NVA (10.1.1.4) through ARS as shown in the above scenarios. Note that next hop shows the ***CSR*** NVA IP and not the Azure Route Server local instances, as **ARS will not be in the data path, it only exchange the BGP routes with the NVA and with the Virtual Network Gateway when the Branch-to-Branch feature enabled.** 


![image](https://user-images.githubusercontent.com/78562461/140263047-3c874910-e538-4af3-9403-edbf82863270.png)



2. **Check ARS (***Routeserver***) advertised routes to ***CSR***:**

```
az network routeserver peering list-advertised-routes --name CSR --routeserver RouteServer --resource-group Route-Server
```

- After peering the ***HUB-SCUS*** with ***Spoke-Vnet***, ARS will advertise the ***Spoke-Vnet*** prefix 10.4.0.0/16 to the ***CSR*** which will advertise it to the ***On-Prem-VNG*** as we will see next.

:exclamation: output below shows the routes advertised from IN_0 but it will be the same for IN_1

![image](https://user-images.githubusercontent.com/78562461/140263490-093b21e4-cde1-496e-8a1e-a6a714d858f4.png)



3. **Check on CSR:**
  
 `#show ip bgp`
	
- Peered ***Spoke-Vnet*** prefix (10.4.0.0/16) has been learned from the ARS local instances 10.1.2.4 and 10.1.2.5 that the ***CSR*** is peering with.

```
CSR#show ip bgp
BGP table version is 9, local router ID is 192.168.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.0.0.0/16      10.0.0.4                               0 65001 i
 *>   10.1.0.0/16      10.1.2.4                               0 65515 i
 *                     10.1.2.5                               0 65515 i
 *>   10.1.10.0/24     10.1.1.1                 0         32768 i
 *    10.2.0.0/16      10.1.2.5                               0 65515 65004 65003 i
 *>                    10.1.2.4                               0 65515 65004 65003 i
 *>   10.4.0.0/16      10.1.2.4                               0 65515 i <========================= Spoke-Vnet Prefix
 *                     10.1.2.5                               0 65515 i

```


4. Check on ***On-Prem-VNG***:

Navigate to Virtual Network Gateways ->  On-Prem-VNG -> Monitoring -> BGP Peers

- Red box shows **Spoke-Vnet** prefix 10.4.0.0/16 is learned from ARS (65515) then CSR (65002), and this will be injected by **On-Prem-VNG** gateway into the **On-Prem-VM** NIC route table as shown below:

👉 ***On-Prem-VNG*** Learned Routes:

![image](https://user-images.githubusercontent.com/78562461/140405102-790a9018-09cb-42cf-aa91-776dc85b5fc6.png)

👉 ***On-Prem-VM*** Effective Routes:

![image](https://user-images.githubusercontent.com/78562461/140405341-663d17b2-822f-407e-9a3c-fd000dab9247.png)


5. ***On-Prem1-VNG*** Gateway learned routes:
 
Navigate to Virtual Network Gateways ->  On-Prem1-VNG -> Monitoring -> BGP Peers
	
- ***HUB-VNG*** gateway learn the route to peered Vnet through the System Route as Global Peering. Due to having the **"Allow Gateway Transit"** and **"Use Remote Gateway"** features enabled on the peering connection between both Vnets, the ***HUB-VNG*** will advertise the ***Spoke-Vnet*** prefix to the ***On-Prem1-VNG***, and so we see ***On-Prem1-VNG*** gateway learned the ***Spoke-Vnet*** prefix with ASN 65004 (***HUB-VNG*** ASN)

![image](https://user-images.githubusercontent.com/78562461/140406079-b7631fba-6e75-40a0-a2ac-0be027bad8a4.png)


## Task 4: Verify Connectivity

🙂 To verify connectivity, ping ***HUB-VM*** (10.1.10.4), ping ***On-Prem-VM*** (10.0.10.4), and ***On-prem1-VM*** (10.2.10.4) from ***Spoke-VM*** (10.4.10.4), all pings will work fine: 

```
azureuser@Spoke-VM:~$ ping 10.1.10.4
PING 10.1.10.4 (10.1.10.4) 56(84) bytes of data.
64 bytes from 10.1.10.4: icmp_seq=1 ttl=64 time=36.2 ms
64 bytes from 10.1.10.4: icmp_seq=2 ttl=64 time=33.1 ms
64 bytes from 10.1.10.4: icmp_seq=3 ttl=64 time=33.0 ms
64 bytes from 10.1.10.4: icmp_seq=3 ttl=64 time=33.1 ms
^C
--- 10.1.10.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 33.013/33.897/36.269/1.377 ms
```

```
azureuser@Spoke-VM:~$ ping 10.0.10.4
PING 10.0.10.4 (10.0.10.4) 56(84) bytes of data.
64 bytes from 10.0.10.4: icmp_seq=1 ttl=63 time=73.2 ms
64 bytes from 10.0.10.4: icmp_seq=2 ttl=63 time=74.5 ms
64 bytes from 10.0.10.4: icmp_seq=3 ttl=63 time=71.6 ms
^C
--- 10.0.10.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 71.698/73.144/74.524/1.154 ms

```
```
azureuser@Spoke-VM:~$ ping 10.2.10.4
PING 10.2.10.4 (10.2.10.4) 56(84) bytes of data.
64 bytes from 10.2.10.4: icmp_seq=1 ttl=64 time=59.3 ms
64 bytes from 10.2.10.4: icmp_seq=2 ttl=64 time=58.1 ms
64 bytes from 10.2.10.4: icmp_seq=3 ttl=64 time=58.5 ms
^C
--- 10.2.10.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 58.128/58.656/59.337/0.542 ms

```


## Scenario 4: Route server multi-region design with IPsec 

In this scenario, we will have another route server in EastUS region, and will configure BGP over IPsec tunnel between the NVAs. We will explore how routes can be exchanged between the NVAs and route servers and how that affects routing within the Vnets and with on-premises networks.
	
For this scenario we will add the following components:

1. ***HUB-EastUS*** Vnet which will have ARS ***Routeserver1***, NVA ***CSR1***, and test VM ***HUB-VM1*** 
2. ***Spoke1-Vnet*** Vnet and will have test VM ***Spoke1-VM***. This Vnet will peer with ***HUB-EastUS*** Vnet
3. Ipsec tunnel with BGP as dynamic routing  between ***CSR*** NVA in ***HUB-SCUS*** Vnet and ***CSR1*** NVA in ***HUB-EastUS*** Vnet, tunnel interface (VTI interface) will be used as BGP peers.
	
After deploying above component the complete diagram will be as below:
	
![image](https://user-images.githubusercontent.com/78562461/145701591-595fdcdd-e2c3-4999-a006-22c912b66dec.png)



## Task1: Create the HUB-EastUS Vnet, HUB1-VM, CSR1 NVA, and ARS Routeserver1
	
- ***HUB-EastU*** Vnet

```
az network vnet create --resource-group Route-Server --name HUB-Eastus --location eastus --address-prefixes 10.3.0.0/16 --subnet-name Subnet-1 --subnet-prefix 10.3.10.0/24 
az network vnet subnet create --address-prefix 10.3.0.0/24 --name CSR1-Subnet --resource-group Route-Server --vnet-name HUB-Eastus
az network vnet subnet create --address-prefix 10.3.1.0/27 --name RouteServerSubnet --resource-group Route-Server --vnet-name HUB-EastUS
```

- ***HUB1-VM***:

```	
az network public-ip create --name HUB1-VMPIP --resource-group Route-Server --location eastus --allocation-method Dynamic
az network nic create --resource-group Route-Server -n HUB1-VMNIC --location eastus --subnet Subnet-1 --private-ip-address 10.3.10.4 --vnet-name HUB-EastUS --public-ip-address HUB1-VMPIP
az vm create -n HUB1-VM -g Route-Server --image UbuntuLTS --admin-username azureuser --admin-password Routeserver123 --size Standard_B1ls --location eastus --nics HUB1-VMNIC
```

- ***CSR1*** NVA in ***HUB-EastUs*** Vnet:

```
az network public-ip create --name CSR1NVA-PublicIP --resource-group Route-Server --idle-timeout 30 --allocation-method Static --location eastus
az network nic create --name CSR1Interface -g Route-Server --subnet CSR1-Subnet --vnet HUB-EastUS --public-ip-address CSR1NVA-PublicIP --ip-forwarding true --private-ip-address 10.3.0.4 --location eastus
az vm create --resource-group Route-Server --location eastus --name CSR1 --size Standard_DS3_v2 --nics CSR1Interface --image cisco:cisco-csr-1000v:17_2_1-byol:17.2.120200508 --admin-username azureuser --admin-password Routeserver123
```
- ARS ***RouteServer1***

```
az network public-ip create --name RouteServer1IP --resource-group Route-Server --version IPv4 --sku Standard --location eastus
subnet_id=$(az network vnet subnet show --name RouteServerSubnet --resource-group Route-Server --vnet-name HUB-EastUS --query id -o tsv) 
az network routeserver create --name RouteServer1 --resource-group Route-Server --hosted-subnet $subnet_id --public-ip-address RouteServer1IP --location eastus
```

## Task2: Create Spoke1-Vnet and Spoke1-VM
	
- ***Spoke1-Vnet***

```	
az network vnet create --resource-group Route-Server --name Spoke1-Vnet  --location eastus --address-prefixes 10.5.0.0/16 --subnet-name Subnet-1 --subnet-prefix 10.5.10.0/24 
```

- ***Spok1-VM***

```	
az network public-ip create --name Spoke1-VMPIP --resource-group Route-Server --location eastus --allocation-method Dynamic
az network nic create --resource-group Route-Server -n Spoke1-VMNIC --location eastus --subnet Subnet-1 --private-ip-address 10.5.10.4 --vnet-name Spoke1-Vnet --public-ip-address Spoke1-VMPIP
az vm create -n Spoke1-VM -g Route-Server --image UbuntuLTS --admin-username azureuser --admin-password Routeserver123 --size Standard_B1ls --location eastus --nics Spoke1-VMNIC
```
## Task3: Create the peering between HUB-EastUS and Spok1-Vnet Vnets

```	
vNet1Id=$(az network vnet show --resource-group Route-Server --name Spoke1-Vnet --query id --out tsv)
vNet2Id=$(az network vnet show --resource-group Route-Server --name HUB-EastUS --query id --out tsv)
	
az network vnet peering create --name Spoke1-To-HubEastus --resource-group Route-Server --vnet-name Spoke1-Vnet --remote-vnet $vNet2Id --allow-vnet-access --use-remote-gateways
az network vnet peering create --name HubEastus-To-Spoke1 --resource-group Route-Server --vnet-name HUB-EastUS --remote-vnet $vNet1Id --allow-vnet-access --allow-gateway-transit
```
## Task4: Establish BGP connection between NVA CSR1 and ARS Routeserver1 
	 
- From ***Routeserver1*** side: create the BGP peering to ***CSR1*** interface 10.3.0.4

```
az network routeserver peering create --name CSR1 --peer-ip 10.3.0.4 --peer-asn 65005 --routeserver RouteServer1 --resource-group Route-Server
```

- From ***CSR1*** side: first we need to know the ARS instances ips to create neighbors in ***CSR1*** BGP process

```
   az network routeserver show --name RouteServer1 --resource-group route-server

            
                             "virtualRouterAsn": 65515,
                                "virtualRouterIps": [
                                    "10.3.1.4", <=======
                                    "10.3.1.5" <=======
```
				    
	
   SSH to ***CSR1***, once login type `conf t` as shown below to get into configuration mode:
```
               CSR#conf t
```
         
	 
   add the following commands one block at a time:

```
router bgp 65005
 bgp log-neighbor-changes
 neighbor 10.3.1.4 remote-as 65515
 neighbor 10.3.1.4 ebgp-multihop 255
 neighbor 10.3.1.5 remote-as 65515
 neighbor 10.3.1.5 ebgp-multihop 255
 !
 address-family ipv4
  neighbor 10.3.1.4 activate
  neighbor 10.3.1.5 activate
 exit-address-family
!
```

   Verify if BGP has established between ARS ***Routeserver1*** and NVA ***CSR1***
	
	CSR1#sh ip bgp summary
	BGP router identifier 192.168.1.4, local AS number 65005
	BGP table version is 10, main routing table version 10
	7 network entries using 1736 bytes of memory
	9 path entries using 1224 bytes of memory
	5/5 BGP path/bestpath attribute entries using 1440 bytes of memory
	4 BGP AS-PATH entries using 128 bytes of memory
	0 BGP route-map cache entries using 0 bytes of memory
	0 BGP filter-list cache entries using 0 bytes of memory
	BGP using 4528 total bytes of memory
	BGP activity 7/0 prefixes, 9/0 paths, scan interval 60 secs
	7 networks peaked at 03:53:47 Nov 30 2021 UTC (00:00:03.013 ago)
	
	Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
	10.3.1.4        4        65515      20      29       10    0    0 00:13:05        2
	10.3.1.5        4        65515      19      29       10    0    0 00:13:05        2
	

   ## Task5: Build IPsec tunnel between CSR NVA in HUB-SCUS Vnet and CSR1 NVA in HUB-EastUS Vnet
       
   - Document the ***CSR1*** NVA public ip ***CSR1NVA-PublicIp*** and the public ip of the ***CSR*** NVA ***CSRPublicIP*** as we will need them to build the Ipsec tunnel in next step:
   
```
az network public-ip show -g  Route-Server -n  CSR1NVA-PublicIP --query "{address: ipAddress}"
az network public-ip show -g  Route-Server -n  CSRPublicIP  --query "{address: ipAddress}"
```

     
   ssh to the ***CSR*** NVA and type `conf t` to get into configuration mode:
```
         CSR#conf t
```

   add the following commands one block at a time, replace ***CSR1NVA-PublicIp*** with the ip you got from above step:
   
```
crypto ikev2 proposal to-csr1-proposal
 encryption aes-cbc-256
 integrity sha1
 group 2
!
crypto ikev2 policy to-csr1-policy
 match address local 10.1.0.4
 proposal to-csr1-proposal
!
crypto ikev2 keyring to-csr1-keyring
 peer CSR1NVA-PublicIp
  address CSR1NVA-PublicIp
  pre-shared-key Routeserver
 !
!
crypto ikev2 profile to-csr1-profile
 match address local 10.1.0.4
 match identity remote address 10.3.0.4 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local to-csr1-keyring
 lifetime 3600
 dpd 10 5 on-demand
!
crypto ipsec transform-set to-csr1-TransformSet esp-gcm 256
 mode tunnel
!
crypto ipsec profile to-csr1-IPsecProfile
 set transform-set to-csr1-TransformSet
 set ikev2-profile to-csr1-profile
!
interface Tunnel12
 ip address 192.168.1.3 255.255.255.255
 ip tcp adjust-mss 1350
 tunnel source 10.1.0.4
 tunnel mode ipsec ipv4
 tunnel destination CSR1NVA-PublicIp
 tunnel protection ipsec profile to-csr1-IPsecProfile
!
router bgp 65002
 neighbor 192.168.1.4 remote-as 65005
 neighbor 192.168.1.4 ebgp-multihop 255
 neighbor 192.168.1.4 update-source Tunnel12
 !
 address-family ipv4
  neighbor 192.168.1.4 activate
  network 192.168.1.3 mask 255.255.255.255
 exit-address-family
!
!add Static route for BGP peer IP over the tunnel
ip route 192.168.1.4 255.255.255.255 Tunnel12
```


❗the above BGP configuration assumes that you have already built the previous scenarios, make sure that complete BGP configuration will be as below:

```
router bgp 65002
 bgp router-id 192.168.1.1
 bgp log-neighbor-changes
 neighbor 10.0.0.4 remote-as 65001
 neighbor 10.0.0.4 ebgp-multihop 255
 neighbor 10.0.0.4 update-source Loopback11
 neighbor 10.1.2.4 remote-as 65515
 neighbor 10.1.2.4 ebgp-multihop 255
 neighbor 10.1.2.5 remote-as 65515
 neighbor 10.1.2.5 ebgp-multihop 255
 neighbor 192.168.1.4 remote-as 65005
 neighbor 192.168.1.4 ebgp-multihop 255
 neighbor 192.168.1.4 update-source Tunnel12
 !
 address-family ipv4
  network 10.1.10.0 mask 255.255.255.0
  network 192.168.1.3 mask 255.255.255.255
  neighbor 10.0.0.4 activate
  neighbor 10.1.2.4 activate
  neighbor 10.1.2.5 activate
  neighbor 192.168.1.4 activate
 exit-address-family
```

- Login to ***CSR1*** and type `conf t` to get into configuration mode:
	
	`CSR1#conf t`
	
	add the following commands one block at a time:
	
```
	crypto ikev2 proposal to-csr-proposal
	 encryption aes-cbc-256
	 integrity sha1
	 group 2
	!
	crypto ikev2 policy to-csr-policy
	 match address local 10.3.0.4
	 proposal to-csr-proposal
	!
	crypto ikev2 keyring to-csr-keyring
	 peer CSRPublicIP
	  address CSRPublicIP
	  pre-shared-key Routeserver
	 !
	!
	!
	crypto ikev2 profile to-csr-profile
	 match address local 10.3.0.4
	 match identity remote address 10.1.0.4 255.255.255.255
	 authentication remote pre-share
	 authentication local pre-share
	 keyring local to-csr-keyring
	 lifetime 3600
	 dpd 10 5 on-demand
	!
	!
	crypto ipsec transform-set to-csr-TransformSet esp-gcm 256
	 mode tunnel
	!
	crypto ipsec profile to-csr-IPsecProfile
	 set transform-set to-csr-TransformSet
	 set ikev2-profile to-csr-profile
	!
	interface Tunnel11
	 ip address 192.168.1.4 255.255.255.255
	 ip tcp adjust-mss 1350
	 tunnel source 10.3.0.4
	 tunnel mode ipsec ipv4
	 tunnel destination CSRPublicIP
	 tunnel protection ipsec profile to-csr-IPsecProfile
	!
	!
	router bgp 65005
	 bgp log-neighbor-changes
	 neighbor 10.3.1.4 remote-as 65515
	 neighbor 10.3.1.4 ebgp-multihop 255
	 neighbor 10.3.1.5 remote-as 65515
	 neighbor 10.3.1.5 ebgp-multihop 255
	 neighbor 192.168.1.3 remote-as 65002
	 neighbor 192.168.1.3 ebgp-multihop 255
	 neighbor 192.168.1.3 update-source Tunnel11
	 !
	 address-family ipv4
	  network 192.168.1.4 mask 255.255.255.255
	  neighbor 10.3.1.4 activate
	  neighbor 10.3.1.5 activate
	  neighbor 192.168.1.3 activate
	 exit-address-family
	!
	!add static route to ARS subnet that point to the default gateway of the CSR-Subnet to avoid recursive routing failure for ARS BGP endpoints learned via BGP
	ip route 10.3.1.0 255.255.255.0 10.3.0.1
	!
	!add static route for BGP peer IP over the tunnel
	ip route 192.168.1.3 255.255.255.255 Tunnel11

```

- Verify that tunnel interface is up from any side
```
CSR1#sh ip interface tunnel11
Tunnel11 is up, line protocol is up
  Internet address is 192.168.1.4/32

CSR#sh ip interface tunnel12
Tunnel12 is up, line protocol is up
  Internet address is 192.168.1.3/32
```
	
- Verify BGP has established over IPsec tunnel

![image](https://user-images.githubusercontent.com/78562461/144916765-475b0172-8cbc-4a43-97c4-efce1884249f.png)

![image](https://user-images.githubusercontent.com/78562461/144917586-b6ad4ffd-9f75-4748-946f-ad3fd4444c68.png)



✔️ From above we see that IPsec and BGP are up between NVAs ***CSR*** and ***CSR1***



## Task6: Verify routing and connectivity

1- Check on NVA ***CSR*** routes
	
👉 From CSR IOS XE:

```
    sh ip bgp
```
	
 - ***CSR*** NVA is learning all the topology routes as follows:
 
   - 10.0.0.0/16 is ***On-Prem-Vnet*** prefix learned directly from ***On-Prem-VNG*** (ASN 65001).
   - 10.1.0.0/16 is ***HUB-SCUS*** Vnet prefix that is advertised by ARS ***Routeserver*** in this Vnet (ASN 65515).
   - 10.2.0.0/16 is ***On-Prem1-Vnet*** that is advertised by ***On-Prem1-VNG*** (ASN 65003) to ***HUB-VNG*** (ASN 65004), then advertised to ARS ***Routeserver*** (65515) then to the ***CSR***.
   - 10.3.0.0/16 is ***HUB-EastUS*** prefix, this route is orginally advertised by ARS ***Routeserver1*** (ASN 65515) to the ***CSR1*** (ASN 65005) then to ***CSR***.
   - 10.4.0.0/16 is ***Spoke-Vnet*** that is peering with ***HUB-SCUS Vnet***, this route is advertised by the ARS ***Routeserver*** (ASN 65515).
   - 10.5.0.0/16 is ***Spoke1-Vnet*** prefix that is advertised by ARS ***Routeserver1*** (ASN 65515) to the ***CSR1*** (ASN 65005) then to ***CSR***.
   - 192.168.1.4 is tunnel interface ip (tunnel 11 in ***CSR1***) that is advertised by ***CSR1*** (ASN 65005). Note that we see 'r>' in front of this route which indicate      RIB failure, it means this route has been learned with lower administrative distance than the one for BGP, and in this case it is because we added static route for BGP peer IP (192.168.1.4) over the tunnel12.
   - 192.168.1.3 is tunnel 12 interface in ***CSR*** NVA that is advertised by the ***CSR*** using Network command.
  
   
   ![image](https://user-images.githubusercontent.com/78562461/144920742-cbe6edee-7169-4f31-8049-e52cb86e467b.png)


👉 From ***CSR*** NICs
           
   You can check from any NIC
	
Navigate to Network Interfaces -> CSROutsideInterface -> Help -> Effective routes
Or:
Navigate to Network Interfaces -> CSRInsideInterface -> Help -> Effective routes
	
- We see that the ***CSROutsideInterface*** and ***CSRInsideInterface*** NICs are not learning ***Spoke1-Vnet*** prefix 10.5.0.0/16 and ***HUB-EastUS*** Vnet prefix 10.3.0.0/16 which are supposed to be programmed in the NIC effective routes by the ARS ***Routeserver***, while it learned all other prefixes in the network diagram which are also programmed by the ARS in the NICs effective routes (10.0.0.0/16, 10.2.0.0/16, 192.68.1.4)! 

![image](https://user-images.githubusercontent.com/78562461/144921682-f8fd3920-1444-469b-876f-696121a839fa.png)


Let check next on the routes ARS ***Routserver*** learned from its peer ***CSR***


2- ARS ***Routeserver*** learned routes
```
az network routeserver peering list-learned-routes --name CSR --routeserver RouteServer --resource-group Route-Server
```

- We see that the ARS is learning only about ***On-Prem-Vnet*** prefix 10.0.0.0/16, ***Subnet-1*** prefix in ***HUB-SCUS*** 10.1.10.0/24, 192.168.1.4 Tunnel11 (VTI interface) in ***CSR1*** NVA, and 192.168.1.3 is Tunnel 12 in ***CSR*** NVA, while it doesn't learn about 10.3.0.0/16 or 10.5.0.0/16, why?
	
❗Note: output here showing routes from IN_0 which is 10.1.2.4 but it will be the same for IN_1 10.1.2.5

![image](https://user-images.githubusercontent.com/78562461/144922817-9a8b0ea4-e7eb-453d-8f60-1780ceafaa22.png)




💡 To know why, let us look at the advertised routes from ***CSR*** to ARS ***Routeserver***, you may use any instance of ARS instances as the BGP neighbor in the following command:

```	
sh bgp neighbors 10.1.2.4 advertised-routes
```
	
- Above shows ***CSR*** is advertising 9 prefixes to the ARS ***Routeserver*** including 10.3.0.0/16 and 10.5.0.0/16, however we only see 10.0.0.0/16, 10.1.10.0/24, 192.168.1.3, and 192.168.1.4 prefixes been learned by the ARS ***Routeserver***, this is due to the BGP loop prevention mechanism, in which the router would drop BGP advertisement when it sees its own AS number in the AS path attribute, and as ARS has ASN of 65515 then any prefix has this ASN in its AS Path attribute will be dropped, and that is the case with 10.3.0.0/16 and 10.5.0.0/16, these two prefixes are originally advertised by the ARS ***Routeserver1*** (65515) then advertised to the ***CSR1*** (65505) then to the ***CSR***, and so when it gets to ARS ***Routeserver*** it will be dropped as ***Routeserver*** will see it is own ASN in the AS Path of this prefix.

![image](https://user-images.githubusercontent.com/78562461/144934937-9e47e3e7-13cc-43b8-a1b6-e65502c01ab4.png)

💡 We can see same behavior between ***CSR1*** and ARS ***Routeserver1*** in which routes advertised by ***CSR1*** to ***Routeserver1*** that has ASN 65515 in its AS Path will be dropped by the ***Routeserver1*** as shown below:
 
-Routes advertised by ***CSR1*** to ***Routeserver1***  
 
 ![image](https://user-images.githubusercontent.com/78562461/144961495-3c189ab4-d847-4ffa-95f5-24490908d410.png)

-Routes learned by ***Routeserver1***:

```
az network routeserver peering list-learned-routes --name CSR1 --routeserver RouteServer1 --resource-group Route-Server
```
	
🕵️‍♀️ Only 4 prefixes have been learned by the ***Routeserver1*** as they don't have ASN 65515 in the AS Path atribute which are: 10.0.0.0/16, 192.168.1.4, 192.168.1.3, and 10.1.10.0/24.
	
![image](https://user-images.githubusercontent.com/78562461/144961794-81cbf655-db44-40d7-bddf-7ea8251acc44.png)

☝️ The consequences for this is that there will be no connectivity between VMs in let say group1 Vnets (***HUB-EastUS*** 10.3.0.0/16 and ***Spoke1-Vnet*** 10.5.0.0/16) and VMs in group 2 Vnets (***HUB-SCUS*** 10.1.0.0/16, ***Spoke-Vnet*** 10.4.0.0/16 and ***On-prem1-Vnet*** 10.2.0.0/16) as the prefixes in each group has been advertised by the ARS with ASN 65515 (either being a prefix for Vnet hosting the ARS, or being a prefix of peered Vnet using ARS in remote Vnet, or on-premises network using ARS to exchange routes with NVA) to the NVA, so when the advertisement reach the other ARS through the NVA (***CSR*** or ***CSR1***) the routes will be dropped and not programmed in the NICs effective route or advertised to on-premises through VPN Gateway.
	
**How to solve this routing issue?**
	
👉 As a workaround solution to this loop prevention feature, we can use the BGP **AS-Override** feature when peering with ARS. AS-Override function causes to replace the AS number of originating router with the AS number of the sending BGP router. This way, ARS on each side will see the ASN of the NVA instead of seeing the ASN of the remote ARS (65515).
	

## Task 7: Configure AS-Override to solve the route propagation issue

- On NVA ***CSR***
	
  - SSH to CSR and type 'conf t' as shown below:
```
      CSR#conf t
```      
	
  - Copy and paste the following commands:
	
	```
	router bgp 65002
	address-family ipv4
	  neighbor 10.1.2.4 as-override
	  neighbor 10.1.2.5 as-override
	 exit-address-family
	```
	
   - Hit `CTRL Z` to get back to enable mode and type `wr` to save the configuration:
   
```	
	CSR#wr
```

- On NVA ***CSR1***
	
  SSH to CSR1 and type 'conf t', then copy and paste the following commands:
  
```
	router bgp 65005
	 address-family ipv4
	  neighbor 10.3.1.4 as-override
	  neighbor 10.3.1.5 as-override
	 exit-address-family
```

 Hit `CTRL Z` to get back to enable mode and type `wr` to save the configuration:
 
```	 
CSR#wr
```


- Check on the ARS learned routes after the AS-Override has been configured on the NVAs
	
1- ARS ***Routeserver***:

```
  az network routeserver peering list-learned-routes --name CSR --routeserver RouteServer --resource-group Route-Server
```

Prior to configuring As-Override, the ***Routeserver*** was only learning 4 prefixes from NVA ***CSR*** which are 192.168.1.4, 192.168.1.3, 10.0.0.0/16, and 10.1.10.0/24, now in addition to those prefixes, it also learned the following prefixes (in red box):

  - 10.1.0.0/16 is ***HUB-SCUS*** Vnet prefix where the ARS ***Routeserver*** reside, 10.4.0.0/16 is the prefix of peered Vnet ***Spoke-Vnet***. Note that ARS will not inject
   those routes in the ***HUB-SCUS*** VMs' NICs or ***Spoke-Vnet*** VMs' NICs with next hop as the ***CSR*** NVA 10.1.1.4, because these are system routes, and ARS will not override them. The ASN in red box is originally 65515 but with AS-Override it has been replaced with ***CSR*** ASN 65002.

 - 10.2.0.0/16 is ***On-Prem1-Vnet*** prefix, we see that the ***Routeserver*** ASN 65515 has been replaced with NVA ***CSR*** ASN 65002 as shown in the red box. Also note this route will not be injected in the ***HUB-SCUS*** VMs' NICs or ***Spoke-Vnet*** VMs' NICs with next hop as ***CSR*** NVA, as this is system route that ARS will not override, also traditionally from BGP route selection perspective, this prefix has been also learned from ***HUB-VNG*** gateway with shorter AS Path, so next hop to this prefix in NICs effective routes will appear as the ***HUB-VNG*** local instances (10.1.5.4 and 10.1.5.5).
	
- 10.3.0.0/16 the ***HUB-EastUS*** Vnet prefix and 10.5.0.0/16 is ***Spoke1-Vnet*** prefix, we can see they are now learned by the ARS ***Routeserver*** and not being dropped after the ASN has been replaced as shown in the red box from 65515 to 65002, and so ARS will inject these routes into NICs effective routes.

![image](https://user-images.githubusercontent.com/78562461/145089454-22ff9391-97c9-4d21-8438-2918c7effcce.png)

2- ARS ***Routeserver1*** learned routes:

- In addition to the 4 prefixes Routeserevr1 was learning (10.0.0.0/16, 192.168.1.4, 192.168.1.3 and 10.1.10.4/24), ***Routeserver1*** now learned the prefixes in red boxes after configuring AS-Override, which are 10.1.0.0/16 ***HUB-SCUS*** Vnet, 10.4.0.0/16 ***Spoke-Vnet***, and 10.2.0.0/16 ***On-Prem1-Vnet*** and those routes will be now injected in the NICs that are using this ARS. ASN in red box is the ***CSR1*** NVA ASN that replaced the 65515 ASN of the ARS.

![image](https://user-images.githubusercontent.com/78562461/145095347-d6234c90-cbeb-4496-8776-efcd21d26ab7.png)

🤝 after this change we have full network connectivity, for example you can ping from ***Spoke1-VM*** to ***Spoke-VM*** and to all other VMs in this topology with no UDR has been used to route the traffic. 

💡 As route tables are not used in this scenario, this design is scalable and so it is suitable for large deployments with multiple spokes and/or multiple on-premises branches across multiple regions, however, traffic goes over IPsec tunnel will have throughput limitation. Next we will explore using Vnet peering instead of IPsec tunnel.

![image](https://user-images.githubusercontent.com/78562461/145097896-435f027c-63af-44e5-9c45-bbcdc3920345.png)


![image](https://user-images.githubusercontent.com/78562461/145623646-60cb00dc-0779-4d53-b165-1a20540579fb.png)
	

## Scenario 5: Route server multi-region design with Vnet peering

In previous scenario we were able to achieve full network connectivity by using ARS and building BGP over IPsec tunnel between the NVAs. However, previous design is useful in scenarios where encryption is needed and bandwidth restrictions are tolerable, as IPsec tunnel or VxLAN tunnel has throughput limitation. In this scenario we will establish BGP over Vnet peering between the NVAs instead of using IPsec tunnel.
	
[Vnet peering](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) provides a low latency and high bandwidth connection that is useful in scenarios such as cross-region data replication and database failover scenarios. 


In this design we will have same components as in scenario 4 but we will have global Vnet peering between ***HUB-EastUS*** and ***HUB-SCUS*** Vnets and will configure BGP over the Vnet peering between the NVAs ***CSR*** and ***CSR1***, we will explore if there will be routing changes when having Vnet peering versus IPsec tunnel between NVAs.

The complete scenario will look as follows:
	
![image](https://user-images.githubusercontent.com/78562461/145637084-4ee5f2e3-c10d-4510-961d-9b4d81c5a9ac.png)



## Task1: Configure Vnet peering between HUB-SCUS and HUB-EastUS Vnets

```
vNet1Id=$(az network vnet show --resource-group Route-Server --name HUB-SCUS --query id --out tsv)
vNet2Id=$(az network vnet show --resource-group Route-Server --name HUB-EastUS --query id --out tsv)

az network vnet peering create --name HUB-SCUS-To-HubEastus --resource-group Route-Server --vnet-name HUB-SCUS --remote-vnet $vNet2Id --allow-vnet-access --allow-forwarded-traffic
az network vnet peering create --name HUBEastus-To-HUB-SCUS --resource-group Route-Server --vnet-name HUB-EastUS --remote-vnet $vNet1Id --allow-vnet-access --allow-forwarded-traffic
	
```
## Task2: configure BGP between CSR and CSR1 NVAs
   
- In the configuration below we will remove the BGP peers established over IPsec tunnel which are (192.168.1.3 and 192.168.1.4) and configure new BGP session between ***CSR*** internal interface 10.1.1.4 and ***CSR1*** interface 10.3.0.4: 
	
	• Login to ***CSR*** NVA, type `conf t` and hit enter to get into the configuration mode then paste the following commands:
```
router bgp 65002
 no neighbor 192.168.1.4 remote-as 65005
 no network 192.168.1.3 mask 255.255.255.255
 neighbor 10.3.0.4 remote-as 65005
 neighbor 10.3.0.4 ebgp-multihop 255
 address-family ipv4
  neighbor 10.3.0.4 activate
! Adding static route to BGP peer to avoid recursive routing failure for CSR1 BGP endpoints learned via BGP
ip route 10.3.0.4 255.255.255.255 10.1.1.1
	
```
	
After pasting above commands make sure that complete BGP configuration should look as below:

```
CSR#sh running-config | sec bgp
router bgp 65002
 bgp router-id 192.168.1.1
 bgp log-neighbor-changes
 neighbor 10.0.0.4 remote-as 65001
 neighbor 10.0.0.4 ebgp-multihop 255
 neighbor 10.0.0.4 update-source Loopback11
 neighbor 10.1.2.4 remote-as 65515
 neighbor 10.1.2.4 ebgp-multihop 255
 neighbor 10.1.2.5 remote-as 65515
 neighbor 10.1.2.5 ebgp-multihop 255
 neighbor 10.3.0.4 remote-as 65005
 neighbor 10.3.0.4 ebgp-multihop 255
 !
 address-family ipv4
  network 10.1.10.0 mask 255.255.255.0
  neighbor 10.0.0.4 activate
  neighbor 10.1.2.4 activate
  neighbor 10.1.2.4 as-override
  neighbor 10.1.2.5 activate
  neighbor 10.1.2.5 as-override
  neighbor 10.3.0.4 activate
 exit-address-family
CSR#
```

• Login to ***CSR1***, type `conf t` and hit enter to get into configuration mode then past the following commands:

```
router bgp 65005
 no neighbor 192.168.1.3 remote-as 65002
 no network 192.168.1.4 mask 255.255.255.255
 neighbor 10.1.1.4 remote-as 65002
 neighbor 10.1.1.4 ebgp-multihop 255
 address-family ipv4
  neighbor 10.1.1.4 activate
! Adding static route to BGP peer to avoid recursive routing failure for CSR BGP endpoints learned via BGP
ip route 10.1.1.4 255.255.255.255 10.3.0.1
```
After pasting above commands make sure that complete BGP configuration should look as below:
	
```
router bgp 65005
 bgp log-neighbor-changes
 neighbor 10.1.1.4 remote-as 65002
 neighbor 10.1.1.4 ebgp-multihop 255
 neighbor 10.3.1.4 remote-as 65515
 neighbor 10.3.1.4 ebgp-multihop 255
 neighbor 10.3.1.5 remote-as 65515
 neighbor 10.3.1.5 ebgp-multihop 255
 !
 address-family ipv4
  network 192.168.1.4 mask 255.255.255.255
  neighbor 10.1.1.4 activate
  neighbor 10.3.1.4 activate
  neighbor 10.3.1.4 as-override
  neighbor 10.3.1.5 activate
  neighbor 10.3.1.5 as-override
 exit-address-family	
```

• Check on BGP session if it has established between the new BGP endpoints 10.1.1.4 and 10.3.0.4 using `sh ip bgp summary` in enable mode:
	
```
CSR#sh ip bgp summary
```
	
We see that BGP session between the new BGP endpoints 10.1.1.4 (***CSR***) and 10.3.0.4 (***CSR1***) has been established, and we don't see the BGP endpoints 192.168.1.4 and 192.168.1.3 in the BGP sessions from either NVA.


![image](https://user-images.githubusercontent.com/78562461/145636461-89f685b9-45e0-4201-bb7f-392dde07e822.png)
![image](https://user-images.githubusercontent.com/78562461/145636541-936c008c-c978-4a0d-bf74-85f8b0e37a88.png)

## Task 3: Verify routing and connectivity


To make it easier to verify connectivity and routing let divide the network in two Sides (**Side1** and **Side2**) as shown below:

![image](https://user-images.githubusercontent.com/78562461/145638689-a49379fd-a03c-482a-92ca-ecdd6690ec72.png)
	
	
1- Check on effective routes for VMs in **Side1**
	
- ***HUB-VM*** effective routes
```
        $ az network nic show-effective-route-table -g Route-Server -n HUB-VMNIC --output table
	Source                 State    Address Prefix    Next Hop Type          Next Hop IP
	---------------------  -------  ----------------  ---------------------  -------------
	Default                Active   10.1.0.0/16       VnetLocal
	VirtualNetworkGateway  Active   10.2.0.0/16       VirtualNetworkGateway  10.1.5.4
	VirtualNetworkGateway  Active   10.2.0.0/16       VirtualNetworkGateway  10.1.5.5
	VirtualNetworkGateway  Active   10.0.0.0/16       VirtualNetworkGateway  10.1.1.4
	VirtualNetworkGateway  Active   10.5.0.0/16       VirtualNetworkGateway  10.1.1.4
	Default                Active   0.0.0.0/0         Internet
	Default                Active   10.4.0.0/16       VNetGlobalPeering
	Default                Active   10.3.0.0/16       VNetGlobalPeering

```

	
- ***Spoke-VM*** effective routes
	
```
	$ az network nic show-effective-route-table -g Route-Server -n Spoke-VMNIC --output table
	Source                 State    Address Prefix    Next Hop Type          Next Hop IP
	---------------------  -------  ----------------  ---------------------  -------------
	Default                Active   10.4.0.0/16       VnetLocal
	VirtualNetworkGateway  Active   10.2.0.0/16       VirtualNetworkGateway  10.1.5.4
	VirtualNetworkGateway  Active   10.2.0.0/16       VirtualNetworkGateway  10.1.5.5
	VirtualNetworkGateway  Active   10.3.0.0/16       VirtualNetworkGateway  10.1.1.4
	VirtualNetworkGateway  Active   10.0.0.0/16       VirtualNetworkGateway  10.1.1.4
	VirtualNetworkGateway  Active   10.5.0.0/16       VirtualNetworkGateway  10.1.1.4
	Default                Active   0.0.0.0/0         Internet
	Default                Active   10.1.0.0/16       VNetGlobalPeering
```

- ***On-Prem1-VM*** effective routes

 > 23.x.x.x is the ***On-Prem1-VNG*** public ip
	
	
```
	$ az network nic show-effective-route-table -g Route-Server -n on-prem1-VMNIC --output table
	Source                 State    Address Prefix    Next Hop Type          Next Hop IP
	---------------------  -------  ----------------  ---------------------  -------------
	Default                Active   10.2.0.0/16       VnetLocal
	VirtualNetworkGateway  Active   10.5.0.0/16       VirtualNetworkGateway  23.X.X.X
	VirtualNetworkGateway  Active   10.1.5.4/32       VirtualNetworkGateway  23.X.X.X
	VirtualNetworkGateway  Active   10.1.0.0/16       VirtualNetworkGateway  23.X.X.X
	VirtualNetworkGateway  Active   10.4.0.0/16       VirtualNetworkGateway  23.X.X.X
	VirtualNetworkGateway  Active   10.3.0.0/16       VirtualNetworkGateway  23.X.X.X
	VirtualNetworkGateway  Active   10.0.0.0/16       VirtualNetworkGateway  23.X.X.X
	Default                Active   0.0.0.0/0         Internet
```
	
- ***On-Prem-VM*** effective routes
	
> 52.x.x.x is the ***On-Prem-VNG*** public ip
	
```
        $ az network nic show-effective-route-table -g Route-Server -n onprem-VMNIC --output table
	Source                 State    Address Prefix    Next Hop Type          Next Hop IP
	---------------------  -------  ----------------  ---------------------  --------------
	Default                Active   10.0.0.0/16       VnetLocal
	VirtualNetworkGateway  Active   10.4.0.0/16       VirtualNetworkGateway  52.X.X.X
	VirtualNetworkGateway  Active   192.168.2.1/32    VirtualNetworkGateway  52.X.X.X
	VirtualNetworkGateway  Active   10.2.0.0/16       VirtualNetworkGateway  52.X.X.X
	VirtualNetworkGateway  Active   192.168.1.1/32    VirtualNetworkGateway  52.X.X.X
	VirtualNetworkGateway  Active   10.1.0.0/16       VirtualNetworkGateway  52.X.X.X
	VirtualNetworkGateway  Active   10.3.0.0/16       VirtualNetworkGateway  52.X.X.X
	VirtualNetworkGateway  Active   10.1.10.0/24      VirtualNetworkGateway  52.X.X.X
	VirtualNetworkGateway  Active   10.5.0.0/16       VirtualNetworkGateway  52.X.X.X
	Default                Active   0.0.0.0/0         Internet
```
	
2- Check on effective routes for VMs in **side2**
	
- ***HUB1-VM*** effective routes

```
        $ az network nic show-effective-route-table -g Route-Server -n HUB1-VMNIC --output table
	Source                 State    Address Prefix    Next Hop Type          Next Hop IP
	---------------------  -------  ----------------  ---------------------  -------------
	Default                Active   10.3.0.0/16       VnetLocal
	Default                Active   10.5.0.0/16       VNetPeering
	VirtualNetworkGateway  Active   10.0.0.0/16       VirtualNetworkGateway  10.3.0.4
	VirtualNetworkGateway  Active   10.2.0.0/16       VirtualNetworkGateway  10.3.0.4
	VirtualNetworkGateway  Active   10.4.0.0/16       VirtualNetworkGateway  10.3.0.4
	Default                Active   0.0.0.0/0         Internet
        Default                Active   10.1.0.0/16       VNetGlobalPeering
```
	
- ***Spoke1-VM*** effective routes
	
```
	$ az network nic show-effective-route-table -g Route-Server -n Spoke1-VMNIC --output table
	Source                 State    Address Prefix    Next Hop Type          Next Hop IP
	---------------------  -------  ----------------  ---------------------  -------------
	Default                Active   10.5.0.0/16       VnetLocal
	Default                Active   10.3.0.0/16       VNetPeering
	VirtualNetworkGateway  Active   10.0.0.0/16       VirtualNetworkGateway  10.3.0.4
	VirtualNetworkGateway  Active   10.2.0.0/16       VirtualNetworkGateway  10.3.0.4
	VirtualNetworkGateway  Active   10.1.10.0/24      VirtualNetworkGateway  10.3.0.4
	VirtualNetworkGateway  Active   10.4.0.0/16       VirtualNetworkGateway  10.3.0.4
	VirtualNetworkGateway  Active   10.1.0.0/16       VirtualNetworkGateway  10.3.0.4
	Default                Active   0.0.0.0/0         Internet
	
```

👉 **From above, we see:**

- All VMs (in both Sides) have routes to all prefixes in the network which is great!
	
- If a VM in **Side1** needs to talk to a VM in **Side2** then it will go through ***CSR*** NVA (10.1.1.4) which reside in **Side1**, and if a VM in **Side2** need to talk to a VM in **Side1** then it goes to ***CSR1*** NVA (10.3.0.4) which reside in ***Side2***, this is similar to scenario 4 (except for traffic between 10.1.0.0/16 and 10.3.0.0/16 as this traffic will go over the global peering). However, if VMs in **side1** try to reach VMs in Side2 or the other direction we will get a loop in this scenario. **Why?**
	
💡 ARS doesn't differentiate between the VMs subnet and the NVA subnet, meaning if ARS learn a route it will programs it for all the VMs in the virtual network including the NVA subnet itself. ***How is that a problem in this scenario?*** let say VM1 tries to talk to VM2, VM1 next hop will be NVA1, NVA1 has next hop to VM2 through NVA1 as well, so will get a loop at NVA1. The same applies if VM2 tries to reach VM1, traffic will be looping at NVA2.
	
**For example:** if ***Spoke-VM*** (10.4.10.4) tries to ping ***Spoke1-VM*** (10.5.10.4), traffic will go to ***CSR*** NVA 10.1.1.4 according to the route table of ***Spoke-VM***, traffic then will reach NVA ***CSR*** which is pointing to ***CSR1*** NVA (10.3.0.4) as next hop to reach ***Spoke1-VM*** in its internal
route table as shown in [1], but the Azure route table for the ***CSR*** NIC has next hop to destination as the NVA ***CSR*** itself as shown in [2], so the traffic to ***Spoke1-VM*** will be sent back to the ***CSR*** NVA (10.1.1.4) and we will get a loop as shown in [3]. We will have similar behavior in the other direction (from ***Spoke1-VM*** to ***Spoke-VM***) in which traffic will get looped at ***CSR1***.

[1]
	
```
CSR#sh ip bgp
BGP table version is 18, local router ID is 192.168.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.0.0.0/16      10.0.0.4                               0 65001 i
 *>   10.1.0.0/16      10.1.2.4                               0 65515 i
 *                     10.1.2.5                               0 65515 i
 *>   10.1.10.0/24     10.1.1.1                 0         32768 i
 *>   10.2.0.0/16      10.1.2.4                               0 65515 65004 65003 i
 *                     10.1.2.5                               0 65515 65004 65003 i
 *>   10.3.0.0/16      10.3.0.4                               0 65005 65515 i
 *>   10.4.0.0/16      10.1.2.4                               0 65515 i
 *                     10.1.2.5                               0 65515 i
 *>   10.5.0.0/16      10.3.0.4                               0 65005 65515 i
	
```

[2]
	
![image](https://user-images.githubusercontent.com/78562461/145663303-520d0cf5-7df1-4fe3-9f33-5b6ba0d83507.png)

[3]

Traceroute and ping from ***Spoke-VM*** to ***Spoke1-VM*** (Side1 to Side2):

```
azureuser@Spoke-VM:~$ ping 10.5.10.4
PING 10.5.10.4 (10.5.10.4) 56(84) bytes of data.
From 10.1.1.4 icmp_seq=1 Time to live exceeded
From 10.1.1.4 icmp_seq=2 Time to live exceeded
From 10.1.1.4 icmp_seq=3 Time to live exceeded
From 10.1.1.4 icmp_seq=4 Time to live exceeded
^C
--- 10.5.10.4 ping statistics ---
4 packets transmitted, 0 received, +4 errors, 100% packet loss, time 3004ms

azureuser@Spoke-VM:~$ traceroute 10.5.10.4
traceroute to 10.5.10.4 (10.5.10.4), 30 hops max, 60 byte packets
 1  10.1.1.4 (10.1.1.4)  38.937 ms  38.917 ms  38.904 ms
 2  10.1.1.4 (10.1.1.4)  40.475 ms  40.462 ms  40.449 ms
 3  10.1.1.4 (10.1.1.4)  43.270 ms  43.257 ms  43.245 ms
 4  10.1.1.4 (10.1.1.4)  45.023 ms  45.011 ms  44.998 ms
```

Traceroute from ***Spoke1-VM*** to Spoke-VM (Side2 to Side1)

```
azureuser@Spoke1-VM:~$ traceroute 10.4.10.4
traceroute to 10.4.10.4 (10.4.10.4), 30 hops max, 60 byte packets
 1  10.3.0.4 (10.3.0.4)  5.622 ms  5.598 ms  5.585 ms
 2  10.3.0.4 (10.3.0.4)  5.573 ms  6.566 ms  6.555 ms
 3  10.3.0.4 (10.3.0.4)  8.674 ms  8.662 ms  8.651 ms
 4  10.3.0.4 (10.3.0.4)  10.306 ms  10.294 ms  10.281 ms
 5  10.3.0.4 (10.3.0.4)  12.658 ms  12.645 ms  12.633 ms
 6  10.3.0.4 (10.3.0.4)  14.189 ms  15.422 ms  15.405 ms
 7  10.3.0.4 (10.3.0.4)  17.601 ms  17.588 ms  18.202 ms
 8  10.3.0.4 (10.3.0.4)  20.144 ms  20.132 ms  20.120 ms
```
> We will get same result when tracing from Spoke1-VM to 10.2.X.X and 10.0.X.X
	
☝️ Why we didn't get this looping issue in scenario 4 even though the NVAs NICs route table are the same as in this scenario?

In scenario 4 we used IPsec encapsulation to prevent packets reaches Azure networking, in this case Azure network will have no visibility about the source and destination, it will only see packets going between the NVAs and so the Azure route table will not affect this traffic and the traffic will take the right address of next hop (which will be the far end NVA) as shown in the NVA internal route table.


**Now how to fix this looping issue in this scenario?**

We will need to create UDRs and assign them to the NVAs subnet to override the Azure route table. We will create two UDRs: **To-Side2** and **To-Side1**:
	
- UDR **To-Side2** will be associated to the ***CSR*** subnet (**internal**), we will point the traffic destined to Side2 prefix 10.5.0.0/16 to go through ***CSR1*** NVA (instead of ***CSR*** NVA).
- UDR **To-Side1** will be associated to the ***CSR1*** subnet (***CSR1-Subnet***), we will point the traffic destined to Side1 prefixes (10.4.0.0/16, 10.2.0.0/16,10.0.0.0/16) to go through ***CSR*** (instead of ***CSR1***).  

- Create UDR **To-Side2** and associate it to the ***CSR*** ***Internal*** subnet:

```
az network route-table create --name To-Side2 --resource-group Route-Server --location southcentralus
az network route-table route create --name Spok1-Vnet --resource-group Route-Server --route-table-name To-Side2 --address-prefix 10.5.0.0/16 --next-hop-type VirtualAppliance --next-hop-ip-address 10.3.0.4
az network vnet subnet update --name internal --vnet-name HUB-SCUS --resource-group Route-Server --route-table To-Side2
```
	
- Create UDR **To-Side1** and associate it to the ***CSR1-Subnet***:

```
az network route-table create --name To-Side1 --resource-group Route-Server --location eastus
az network route-table route create --name Spok-Vnet --resource-group Route-Server --route-table-name To-Side1 --address-prefix 10.4.0.0/16 --next-hop-type VirtualAppliance --next-hop-ip-address 10.1.1.4
az network route-table route create --name On-prem1-Vnet --resource-group Route-Server --route-table-name To-Side1 --address-prefix 10.2.0.0/16 --next-hop-type VirtualAppliance --next-hop-ip-address 10.1.1.4
az network route-table route create --name On-Prem-Vnet --resource-group Route-Server --route-table-name To-Side1 --address-prefix 10.0.0.0/16 --next-hop-type VirtualAppliance --next-hop-ip-address 10.1.1.4
az network vnet subnet update --name CSR1-subnet --vnet-name HUB-EastUS --resource-group Route-Server --route-table To-Side1
	
```


After applying the UDRs, ***Spoke-VM*** can reach ***Spoke1-VM*** and vice versa:

```
azureuser@Spoke-VM:~$ traceroute 10.5.10.4
traceroute to 10.5.10.4 (10.5.10.4), 30 hops max, 60 byte packets
 1  10.1.1.4 (10.1.1.4)  37.028 ms  37.052 ms  36.993 ms
 2  10.3.0.4 (10.3.0.4)  73.435 ms  73.422 ms  73.407 ms
 3  10.5.10.4 (10.5.10.4)  75.200 ms  75.187 ms *


azureuser@Spoke1-VM:~$ traceroute 10.4.10.4
traceroute to 10.4.10.4 (10.4.10.4), 30 hops max, 60 byte packets
 1  10.3.0.4 (10.3.0.4)  2.023 ms  1.975 ms  1.963 ms
 2  10.1.1.4 (10.1.1.4)  36.549 ms  36.535 ms  38.433 ms
 3  * 10.4.10.4 (10.4.10.4)  73.625 ms *
	
```

Also ***Spoke1-VM*** can reach all other VMs in **Side2** (***On-Prem1-VM*** and ***On-Prem-VM***).
	
> The use of Vnet peering in this design eliminates the throughput limitation associated with IPsec tunnel that is used in scenario 4, however, unlike scenario 4, this design is not as scalable as scenario 4 if more Vnets are introduced, as the routes in the UDR associated with NVAs subnet are statically configured.

	
## Scenario 6: On-premises internet breakout through Azure
	
In this scenario we will explore how Azure Route Server can be used to force tunnel internet bound traffic from on-premises network as well as VMs in Vnets through an NVA deployed in Azure. In this scenario the NVA CSR will advertise default route through BGP and we will check on routing and connectivity post the advertising.

We will test the internet breakout (force tunneling) on the following part of the network:

![image](https://user-images.githubusercontent.com/78562461/148292130-80765a33-5b0d-4406-b9ef-a851acdb53bc.png)

	
To setup this scenario, we will do the following configuration changes:


1- As the NVA ***CSR*** will advertise the default route to the network, we will lose connectivity to the CSR also the VPN tunnel between the ***CSR*** and ***On-Prem-VNG*** will go down, so to fix this issue we will:
	
 - Create Azure Bastion in Vnet ***HUB-SCUS*** to access the ***CSR*** and ***HUB-VM*** after advertising the default route. We will use ***HUB-VM*** to SSH to other VMs in the network using their private ips as we have full network connectivity within the private network, so we can verify the connectivity to the internet from those VMs after CSR advertises the default route.
	
 - Create a UDR and assign it to the **External** subnet where the outside interface **CSROutsideInterface** reside, the UDR will have a route to the ***On-Prem-VNG*** public ip with next hop type **Internet**. This interface has a public ip that is used to build the VPN tunnel with ***On-Prem-VNG*** and so it will need the internet connectivity to build the tunnel.
	
2- On the ***CSR*** NVA:
   - Remove BGP peering between the NVAs ***CSR*** and ***CSR1***
   - Remove the as-override configuration with Route Server BGP endpoints   
   - Advertise default route through BGP
   - Configure nat to nat private ips of the VMs to the public ip of the ***CSR*** interface **CSROutsideInterface**

## Task 1: Install traceroute tool

To validate and to troubleshoot connectivity, install traceroute tool on the following VMs: ***HUB-VM***, ***Spoke-VM***, ***On-Prem-VM***, ***On-Prem1-VM***.
	
- Login to each VM and paste the following command that will install the traceroute tool:
	
```
sudo apt-get update &&sudo apt-get install traceroute -y
```
	
## Task 2: Configure Azure Bastion

```
az network public-ip create --resource-group Route-Server --name BastionPIP --sku Standard --location southcentralus
az network vnet subnet create --address-prefix 10.1.6.0/26 --name  AzureBastionSubnet --resource-group Route-Server --vnet-name HUB-SCUS
az network bastion create --name Bastion --public-ip-address BastionPIP --resource-group Route-Server --vnet-name HUB-SCUS --location southcentralus
```
## Task 3: Configure the UDR and assign it to the CSR **External** subnet

- Get the ***On-Prem-VNG*** public ip as we will need it to configure the route in the route table: 

```
az network public-ip show -g Route-Server -n Azure-VNGpubip --query "{address: ipAddress}"  
```

- Create the UDR and replace the X.X.X.X with the ip you got from above command:

```
az network route-table create --name Outside-Interface-RT --resource-group Route-Server --location southcentralus
az network route-table route create --name To-On-Prem-VNG --resource-group Route-Server --route-table-name Outside-Interface-RT --address-prefix X.X.X.X/32 --next-hop-type Internet
az network vnet subnet update --name External --vnet-name HUB-SCUS --resource-group Route-Server --route-table Outside-Interface-RT
```
	





