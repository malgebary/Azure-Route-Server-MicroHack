# **Azure Route Server MicroHack**

# Contents

[Introduction](#introduction)

[Objectives](#objectives)

[Lab](#lab)

[Notes](#Notes)

[Scenario 1: Connect on-premise network with CSR NVA in Azure Vnet and use ARS to inject on-premise routes into the Vnet](#Scenario-1-Connect-on-premise-network-with-CSR-NVA-in-Azure-Vnet-and-use-ARS-to-inject-on-premise-routes-into-the-Vnet)

[Scenario 2: Route exchange between Virtual Network Gateway and CSR NVA](#Scenario-2-Route-exchange-between-Virtual-Network-Gateway-and-CSR-NVA)

[Scenario 3: Vnet peering and ARS](#Scenario-3-Vnet-peering-and-ARS)

# Introduction

Azure Route Server ([ARS](https://docs.microsoft.com/en-us/azure/route-server/overview)) enables dynamic routing between Network Virtual Appliance (NVA) and Virtual Network (Vnet) by supporting Border Gateway Protocol (BGP). Before ARS, users had to create a User Defined Route (UDR) to NVA, and need to manually update the routing table on the NVA whenever Vnet addresses get updated. With the introduction of Azure Route Server, users can now inject a similar route without having to create the UDRs, or manually update the NVA with new Vnet prefixes.

ARS enables the NVA to advertise routes to other endpoints in the local/peered Vnet; as well as endpoints defined in on-premises networks connected via ExpressRoute or VPN.  Additionally, ARS allow transit between Virtual Network Gateways (ExpressRoute and VPN) which was not possible before. 

In this MicroHack we will explore some routing scenarios that shows how ARS can be utilized to simplify configuration, management, and deployment of the NVAs in the Virtual Network.

# Objectives:

After completing this MicroHack you will:

- Know how to build Azure Route Server in a Vnet and connect it to an NVA and/or virtual network gateway.
- Understand the new routing capabilities introduced by ARS and how to interpret the route table after deploying the ARS in the Vnet.

# Lab:

Azure Cli will be used to build this Lab. the complete lab consist of:  

  1. Four Vnets: ***On-Prem-Vnet***, ***On-prem1-Vnet***, ***HUB-SCUS***, and ***Spoke-Vnet***.
  2. Three Virtual Network Gateways:***On-Prem-VNG***, ***On-Prem1-VNG***, and ***HUB-VNG***.
  3. Five VMs: ***On-Prem-VM***, ***On-Prem1-VM***, ***HUB-VM***, ***CSR***, and ***Spoke-VM***.
  4. Azure Route Sever: ***RouteServer***.
	
  
At the end of the Lab the deployment will look like this:


![image](https://user-images.githubusercontent.com/78562461/140991540-2ea50ef8-1310-4abf-bf7d-5d8bfb4791dd.png)

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

## Scenario 1: Connect on-premise network with CSR NVA in Azure Vnet and use ARS to inject on-premise routes into the Vnet

In this scenario you will connect on-premise network (***On-Prem-Vnet***) with NVA in the ***HUB-SCUS*** Vnet, and we will observe the routing table before and after deploying ARS to the ***HUB-SCUS*** Vnet.

This scenario consist of:

1. ***On-prem-Vnet*** to simulate on-premise network, Vpn Gateway ***On-Prem-VNG*** to simulate gateway on the on-premise, and an VM ***On-Prem-VM***.
2. ***HUB-SCUS*** Vnet represent Azure side that has NVA (***CSR***), VM (***HUB-VM***), and ARS (***RouteServer***).
3. NVA ***CSR*** with two Nics, one external (CSROutsideInterface) and one Internal (CSRInsideInterface), external interface will be used as the source for Ipsec tunnel, while internal interface will be used as BGP peer with ARS.
4. We will use loopback interface on the ***CSR*** (Loopback11) as update source for the BGP peering with ***On-Prem-VNG***, while will use the internal interface of the ***CSR*** as the BGP peer with ARS instances.
	

After deploying above, the diagram will look like following:

![image](https://user-images.githubusercontent.com/78562461/140993322-3bf5a79a-c6f3-4b9f-ad76-a82c1a3a78c5.png)

## Task1: Deploy On-Prem-Vnet and HUB-SCUS Vnet without the ARS 

-**Create the resource group:**

    az group create --name Route-Server --location centralus

-**Create the On-Premise Network:**

**On-prem-Vnet**:

    az network vnet create --resource-group Route-Server --name On-Prem-Vnet --location northcentralus --address-prefixes 10.0.0.0/16 --subnet-name Subnet-1 --subnet-prefix 10.0.10.0/24 
    az network vnet subnet create --address-prefix 10.0.0.0/27 --name GatewaySubnet --resource-group Route-Server --vnet-name On-Prem-Vnet


**On-Prem-VM (Linux):**

    az network public-ip create --name On-PremVMIP --resource-group Route-Server --location northcentralus --allocation-method Dynamic
    
    az network nic create --resource-group Route-Server -n OnPrem-VMNIC --location northcentralus --subnet Subnet-1 --private-ip-address 10.0.10.4 --vnet-name On-Prem-Vnet --public-ip-address  On-PremVMIP
    
    az vm create -n On-Prem-VM -g Route-Server --image UbuntuLTS --admin-username azureuser --admin-password Routeserver123 --size Standard_B1ls --location northcentralus --private-ip-address 10.0.10.4 --nics OnPrem-VMNIC

**On-Prem-VNG:**

**Note:** The VPN GW will take 20+ minutes to deploy

    az network public-ip create --name Azure-VNGpubip --resource-group Route-Server --allocation-method Dynamic --location northcentralus

    az network vnet-gateway create --name On-Prem-VNG --public-ip-address Azure-VNGpubip --resource-group Route-Server --vnet On-Prem-Vnet --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --asn 65001 --bgp-peering-address 10.0.0.4 --location northcentralus


**-Create the Azure HUB Network:**

**HUB-SCUS Vnet:**

    az network vnet create --resource-group Route-Server --name HUB-SCUS --location southcentralus --address-prefixes 10.1.0.0/16 --subnet-name Subnet-1 --subnet-prefix 10.1.10.0/24 
    az network vnet subnet create --address-prefix 10.1.0.0/24 --name External --resource-group Route-Server --vnet-name HUB-SCUS
    az network vnet subnet create --address-prefix 10.1.1.0/24 --name Internal --resource-group Route-Server --vnet-name HUB-SCUS
    az network vnet subnet create --address-prefix 10.1.2.0/27 --name RouteServerSubnet --resource-group Route-Server --vnet-name HUB-SCUS
    az network vnet subnet create --address-prefix 10.1.5.0/27 --name GatewaySubnet --resource-group Route-Server --vnet-name HUB-SCUS

**HUB-VM (Linux):**

    az network public-ip create --name HUB-VMPIP --resource-group Route-Server --location southcentralus --allocation-method Dynamic
    az network nic create --resource-group Route-Server -n HUB-VMNIC --location southcentralus --subnet Subnet-1 --private-ip-address 10.1.10.4 --vnet-name HUB-SCUS --public-ip-address  HUB-VMPIP
    az vm create -n HUB-VM -g Route-Server --image UbuntuLTS --admin-username azureuser --admin-password Routeserver123 --size Standard_B1ls --location southcentralus --private-ip-address 10.1.10.4 --nics HUB-VMNIC

**CSR NVA:**

Here we are going to use Cisco CSR1000v as NVA. This NVA will have two Nics, one external (CSROutsideInterface) and one Internal (CSRInsideInterface). Before deploying the ***CSR***, you need to accept license agreement:

    az vm image terms accept --urn cisco:cisco-csr-1000v:17_2_1-byol:latest 

    az network public-ip create --name CSRPublicIP --resource-group Route-Server --idle-timeout 30 --allocation-method Static --location southcentralus
    az network nic create --name CSROutsideInterface -g Route-Server --subnet External --vnet HUB-SCUS --public-ip-address CSRPublicIP --ip-forwarding true --private-ip-address 10.1.0.4 --location southcentralus
    az network nic create --name CSRInsideInterface -g Route-Server --subnet Internal --vnet HUB-SCUS --ip-forwarding true --private-ip-address 10.1.1.4 --location southcentralus
    az vm create --resource-group Route-Server --location southcentralus --name CSR --size Standard_DS3_v2 --nics CSROutsideInterface CSRInsideInterface --image cisco:cisco-csr-1000v:17_2_1-byol:17.2.120200508 --admin-username azureuser --admin-password Routeserver123


- **Build Ipsec tunnel between ***CSR*** NVA (respresent Azure side) and ***On-Prem-VNG*** gateway (represent On-Premise side):**

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
Type `exit` multiple times until the prompt shows `csr#` and save the configuration with `wr` as shown below:

    CSR# wr

• Now we need to create the tunnel (connection) from the on-premise side, and for that we will create the Local Network Gateway (LNG) and then the connection:

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

![image](https://user-images.githubusercontent.com/78562461/139952513-ba1af818-1b2c-4801-a9f9-343f0e8452ce.png)



💡 Traditionally, to have ***HUB-VM*** able to ping ***On-Prem-VM*** we would need to add UDR on the ***subnet-1*** (where ***HUB-VM*** reside) to direct traffic destined to 10.0.10.0/24 (or in general to remote Vnet 10.0.0.0/16) to the ***CSR*** internal interface 10.1.1.4. But with ARS there is no need for UDR, ARS will inject the route it learns from ***CSR*** (NVA) automatically. We will see this next 👇.



 
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
	 
### 1. RouteServer Routes:

   :exclamation: Note: we can check on Learned/advertised routes per peer at a time, which is here we have one peer (***CSR***):
   
   :point_right: Learned Routes from ***CSR***:

      az network routeserver peering list-learned-routes --name CSR --routeserver RouteServer --resource-group Route-Server
	
**·** We see all routes have Next Hop 10.1.1.4 which is the ***CSR*** internal (LAN) interface that the ARS ***RouteServer*** is peering with. Pay attention to the asPath, the one with 65002-65001 shows this route actually been advertised from (***On-Prem-VNG***) which is here 10.0.0.0/16, while route 10.1.10.0/24 with asPath 65002 is a route been advertised by the ***CSR*** itself .

Note that the ARS has two instances IN_0 (10.1.2.4) and IN_1 (10.1.2.5) and we will see same routes learned the same way for the other instance.

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


### 2. CSR (NVA) Routes:
	
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
	
- Route with Next Hop 10.1.2.4 and 10.1.2.5 with ASN 65515 is a route learned from ARS, and here it shows the ***CSR*** learned the ***HUB-SCUS*** Vnet prefix (10.1.0.0/16)
  from ARS and it will send it then through eBGP over Ipsec to ***On-prem-VNG***, this way ***On-Prem-Vnet*** will learn the ***HUB-SCUS*** prefix. Note that the route
  10.1.0.0/16 is learned from the two Route Server instances 10.1.2.4 and 10.1.2.5, but the path through 10.1.2.4 has been chosen as best (**>**) based on [BGP Best Path Selection Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html).

	
- Route with Next Hop 10.0.0.4 with ASN 65001 is a route received from ***On-Prem-VNG***, which is here 10.0.0.0/16, ***CSR*** will send it to ARS (as we saw above in ARS learned routes) through eBGP, and ARS in turn will advertise it to the ***HUB-SCUS*** Vnet (will see that later).
	
- Route with Next Hop 10.1.1.1 and no ASN, but with weight 32768 is a route advertised by the ***CSR*** itself, here it is the route 10.1.10.0/24 which refer to ***HUB-VM*** subnet (***Subnet-1***).


:point_right: Check on effective routes on the ***CSR*** NICs, either by using the portal or by the following Cli commands:
  
   ```
 az network nic show-effective-route-table -g Route-Server -n CSROutsideInterface --output table
 az network nic show-effective-route-table -g Route-Server -n CSRInsideInterface --output table
   ```
   
* Both NICs return the below table, notice that 10.0.0.0/16 (***On-Prem-Vnet***) got injected automatically (by ARS) to the ***CSR*** NICs' route table with Next Hop Ip as the ***CSR*** LAN Interface and Next Hop Type as Virtual Network Gateway, so traffic will be directed to the ***CSR*** directly. This shows that **ARS is not in the data path**, it only exchange BGP routes with NVA and program the routes it learns in the NICs' route table.

:exclamation: Note that we don't see 10.1.10.0/24 route got programmed in the Nics' route table which is ***Subnet-1*** prefix that belong to the Vnet (***HUB-SCUS***), even though it is learned by the ARS from the ***CSR*** as shown in ARS learned routes above, why?

:bulb: Because ARS will not program any route equal or more specific to the Vnet/Subnets prefix into the Vnet, meaning that we cannot override the system Vnet routes.

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

### 3. Effective Route on HUB-VM: 

 Use Azure Cli:
 
     az network nic show-effective-route-table -g Route-Server -n HUB-VMNIC --output table
	
We see ***On-Prem-Vnet*** prefix 10.0.0.0/16 is injected automatically by ARS in the NIC route table with Next Hop Ip as the ip of the LAN interface of the ***CSR***:

![image](https://user-images.githubusercontent.com/78562461/140001830-19a3fba2-aabf-4806-979b-70a5996bcd49.png)

### 4. Learned routes by the On-Prem-VNG gateway:
	
- Check on learned routes from the portal by navigating to Virtual Network Gateways -> On-Prem-VNG -> Monitoring -> BGP Peers or use Azure Cli `az network vnet-gateway list-learned-routes -g Route-Server -n On-Prem-VNG`.
	
- Next hop 192.168.1.1 is the Loopback 11 that has been used as update source for BGP in the ***CSR***, this address has been added to the LNG of the ***On-Prem-VNG*** so it knows how to get to this BGP peer ip, and also added 192.168.2.1 (Tunnel11 ip in the ***CSR***) to the LNG, that is why **Origin** to these two addresses shows as **Network**. 
	
- Route 10.1.10.0/24 is the ***subnet-1*** prefix in the ***HUB-SCUS*** Vnet that is advertised by Network command in the ***CSR***, and so we see the AS path has 65002 which is ***CSR*** ASN. 
	
- Route 10.1.0.0/16 is the ***HUB-SCUS*** Vnet prefix, learned through the ***CSR***(192.168.1.1), it is advertised originally from the ARS as illustrated by the As path 65002-65515, in which 65515 is the ARS ASN.
	
![image](https://user-images.githubusercontent.com/78562461/140002829-5b74b986-8ea9-41ff-82d6-dffcb796705d.png)




### 4. Effective Route on On-Prem-VM:
	
- Use Azure Cli  `az network nic show-effective-route-table -g Route-Server -n OnPrem-VMNIC --output table`
	
- We see that all routes learned by the ***On-Prem-VNG*** gateway is injected to the NIC with the next hop as the public ip of the ***On-prem-VNG*** gateway (20.88.39.59 in this example).

```
$ az network nic show-effective-route-table -g Route-Server -n OnPrem-VMNIC --output table
Source                 State    Address Prefix    Next Hop Type          Next Hop IP
---------------------  -------  ----------------  ---------------------  -------------
Default                Active   10.0.0.0/16       VnetLocal
VirtualNetworkGateway  Active   10.1.0.0/16       VirtualNetworkGateway  20.88.39.59
VirtualNetworkGateway  Active   192.168.1.1/32    VirtualNetworkGateway  20.88.39.59
VirtualNetworkGateway  Active   192.168.2.1/32    VirtualNetworkGateway  20.88.39.59
VirtualNetworkGateway  Active   10.1.10.0/24      VirtualNetworkGateway  20.88.39.59
Default                Active   0.0.0.0/0         Internet

```

👉 The completed BGP path between ***On-Prem-VM*** and ***HUB-VM*** is shown below:

![image](https://user-images.githubusercontent.com/78562461/140006065-29623306-8c94-4483-90c0-38cb5e6e5f0a.png)

🙂 From above we can see that ***HUB-VM*** has route now to ***On-Prem-VM*** after introducing ARS with no UDR needed, and so ping will work:


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

In this scenario we will add the following components:

1- ***HUB-VNG*** which represent virtual network gateway in the ***HUB-SCUS*** Vnet in active-active mode.

2- New Vnet called ***On-Prem1-Vnet*** represent another on-premise network, this Vnet will have ***On-Prem1-VNG*** which represent On-premise gateway, and will also have testing VM ***On-Prem1-VM***.

3- Ipsec site to site single tunnel between the two virtual network gateways (***Hub-VNG*** and ***On-Prem1-VNG***) using BGP as the dynamic routing protocol.

Once above get deployed the complete diagram will look as below:

![image](https://user-images.githubusercontent.com/78562461/141018182-f6b4e14e-cc5d-42d7-aef6-dd9ccc47ec51.png)


## Task1: Deploy the ***HUB-VNG*** virtual network gateway in the ***HUB-Vnet***

	
- Create BGP enabled gateway in active-active mode, it takes 20+ minutes to get deployed:
	
:exclamation:To allow route transit between ARS and virtual network gateway, the latter need to be in [active-active](https://docs.microsoft.com/en-us/azure/route-server/expressroute-vpn-support#how-does-it-work) mode.
	
```	
az network public-ip create --name HUB-VNG-PIP1 --resource-group Route-Server --allocation-method Dynamic --location southcentralus
az network public-ip create --name HUB-VNG-PIP2 --resource-group Route-Server --allocation-method Dynamic --location southcentralus 
az network vnet-gateway create --name HUB-VNG --public-ip-address HUB-VNG-PIP1 HUB-VNG-PIP2 --resource-group Route-Server --vnet HUB-SCUS --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --asn 65004  --location southcentralus
```

## Task2: Build the on-premise network (***On-prem1-Vnet***) and its components
	
	
- Create ***On-Prem1-Vnet***:
```	
az network vnet create --resource-group Route-Server --name On-Prem1-Vnet --location centralus --address-prefixes 10.2.0.0/16 --subnet-name Subnet-1 --subnet-prefix 10.2.10.0/24 
az network vnet subnet create --address-prefix 10.2.2.0/27 --name GatewaySubnet --resource-group Route-Server --vnet-name On-Prem1-Vnet
```

- Create the ***On-Prem1-VM***:
```	
az network public-ip create --name On-Prem1-VMPIP --resource-group Route-Server --location centralus --allocation-method Dynamic
az network nic create --resource-group Route-Server -n On-Prem1-VMNIC --location centralus --subnet Subnet-1 --private-ip-address 10.2.10.4 --vnet-name On-Prem1-Vnet --public-ip-address On-Prem1-VMPIP
az vm create -n On-Prem1-VM -g Route-Server --image UbuntuLTS --admin-username azureuser --admin-password Routeserver123 --size Standard_B1ls --location centralus --nics On-Prem1-VMNIC
```

- Create the ***On-Prem1-VNG***. It takes 20+ minutes to get deployed:

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
       
You can check on same from portal, navigate to Virtual Network Gateway -> HUB-VNG -> Settings -> Configuration, check on Default Azure BGP peer ip address, and you can also find HUB-VNG-PIP1 you got from earlier here:

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
	
Navigate to Network interfaces -> OnPrem-VMNIC -> Help -> Effective routes:
	
- 10.1.0.0/16 is the ***HUB-SCUS*** Vnet prefix.
- 1.2.0.0/16 is the ***On-Prem1-Vnet*** prefix (where the destination ***On-Prem1-VM*** is located) so we can see the source VM learned the route to destination VM.
- 192.168.1.1 is the ***CSR*** BGP peer ip and 192.168.2.1 is ***CSR*** Tunnel11 ip.
- 10.1.10.0/24 is the ***Subnet-1*** in ***HUB-SCUS*** Vnet (this prefix is advertised beside the ***HUB-SCUS*** Vnet prefix as the ***CSR*** advertised this route using Network command).

All those routes are showing **Next Hop Type** as **Virtual Network Gateway** which refer to the ***On-Prem-VNG*** gateway. Next we will check how this gateway learned those routes.

![image](https://user-images.githubusercontent.com/78562461/141040848-b54ed58c-c9c8-4558-b440-0ab89fb90bd1.png)


2. **Check ***On-Prem-VNG*** gateway:** 
	
Navigate to Virtual Network Gateways -> On-Prem-VNG -> Monitoring -> BGP Peers:
	
👉 **BGP Peers:**
	
- The local gateway BGP ip 10.0.0.4 is peering successfully with ***CSR*** BGP peer ip 192.168.1.1. Let check next on routes learned due to this peering. 
	
	![image](https://user-images.githubusercontent.com/78562461/140189765-fb819b17-4f70-4ff9-a1bf-e98026996a19.png)

	
	
👉 **Learned Routes:** 
	
- 10.2.0.0/16 is the Vnet prefix of the destination VM ***On-Prem1-VM***. Check the AS Path to see how the gateway learned the route to the destination: ASN 65003 refer to the destination gateway (***On-Prem1-VNG***) which originally advertised this route to the ***HUB-VNG*** gateway that has the ASN 65004 through eBGP peering, this route then advertised to the Route Server as **Branch-to-Branch** feature is enabled, so we see the ASN 65515 in the AS Path, then Route Server advertised it to the ***CSR*** which has the ASN 65002 and ***CSR*** then advertise it down to the ***On-Prem-VNG***.
	
- 10.1.0.0/16 is the ***HUB-SCUS*** Vnet prefix that is learned by the gateway from ***CSR*** ASN 65002 through eBGP peering over Ipsec, note that this prefix originally advertised by the Route Server through eBGP peering to the ***CSR***, and that why we see 65002-65515 in the AS path.
	
- 10.1.10.0/24 is the ***Subnet-1*** prefix in the ***HUB-SCUS*** Vnet, note that the AS Path shows only 65002 which is the ASN of ***CSR*** as this route is advertised using Network command in the ***CSR*** BGP configuration as mentioned earlier (CSR BGP configuration is shown below).

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
	
- 10.0.0.0/16 is the ***On-Prem-Vnet*** prefix (source prefix), it shows it got learned from 10.0.0.4 which is the BGP peer ip of ***On-prem-VNG***, we can also see that from the ASN 65001 which refer to ***On-Prem-VNG***.
	
- 10.2.0.0/16 is the ***On-prem1-Vnet*** (destination prefix) that is learned from the Route Server instances 10.1.2.4 and 10.1.2.5.Look at the AS Path to track how the ***CSR*** learned this prefix: ASN 65003 refer to ***On-prem1-VNG*** that own the prefix 10.2.0.0/16, then 65004 is the ASN of ***HUB-VNG*** gateway which learned this prefix from ***On-Prem1-VNG***, Route Server with ASN 65515 then learned this prefix and advertised it to ***CSR*** which in turn advertised it down to ***On-Prem-VNG*** as we saw earlier.  
	
- 10.1.0.0/16 is the ***HUB-SCUS*** prefix where the ***CSR*** reside, learned through the Route Server instances, the ***CSR*** will advertise it then to the ***On-Prem-VNG*** as we saw from ***On-Prem-VNG*** gateway learned routes above.

![image](https://user-images.githubusercontent.com/78562461/140201163-4eb56756-dcc9-4663-ae59-7bae588e1b44.png)



4. **Check on ARS (RouteServer) routes:**
	
👉 **Check on routes learned from CSR NVA:**

    az network routeserver peering list-learned-routes --name CSR --routeserver RouteServer --resource-group Route-Server
	
- Both ARS instances IN_0 (10.1.2.5) and IN_1 ( 10.1.2.4) learn the same routes from the ***CSR***. 
	
- 10.1.10.0/24 is the ***Subnet-1*** prefix that the ***CSR*** (ASN 65002) advertised manually using Network command, 10.0.0.0/16 is ***On-Prem-Vnet*** prefix that is learned from ***On-Prem-VNG*** (ASN 65001) and the ***CSR*** (65002) advertised it then to ARS. 


![image](https://user-images.githubusercontent.com/78562461/140200447-531d93bd-8197-413a-820f-18e1578ea3e0.png)

👉 **Check on Routes advertised from ARS to the CSR:**

    az network routeserver peering list-advertised-routes --name CSR --routeserver RouteServer --resource-group Route-Server
	
- Although ARS will not inject the ***HUB-Vnet*** prefix 10.1.0.0/16 into the NICs effective routes as mentioned above, it will advertise this prefix to the ***CSR*** over eBGP so ***CSR*** can advertise it down to ***On-Prem-VNG*** gateway (On premise connected gateway over Ipsec).
	
- 10.2.0.0/16 is the ***On-Prem1-Vnet*** prefix that is learned from ***HUB-VNG (65004)*** which has learned it from ***On-Prem1-VNG*** (65003).

![image](https://user-images.githubusercontent.com/78562461/140241249-f9037e9e-1847-4a4b-be9b-ea283981498f.png)

	
5. **Check on HUB-VNG:**
	
👉 **BGP Peers:** Navigate to Virtual Network Gateways -> HUB-VNG -> Monitoring -> BGP peers
	
- Peer addresses in yellow boxes with ASN 65515 are the Route Server BGP peer ips peering with both ***HUB-VNG*** gateway instances (10.1.5.5 and 10.1.5.4) and all are successfully connected.
	
- Peer address 10.2.2.30 in red box with ASN 65003 is the BGP peer ip of the ***On-Prem1-VNG*** gateway, note that we only see it in "connected" state with instance 10.1.5.4 as we have built only one tunnel between the two gateways and we used this ***HUB-VNG*** BGP ip 10.1.5.4 to peer with ***On-Prem1-VNG*** gateway.


![image](https://user-images.githubusercontent.com/78562461/140241905-f02546c1-46dd-41fb-b56e-6dc38e156da4.png)

👉 **Learned Routes:**
	
:exclamation: Note that results here will show learned routes for the two gateway instances of the ***HUB-VNG*** gateway, however, as we built only one tunnel with ***On-Prem1-VNG*** results below are filtered for the one instance used to build this tunnel, which is local address 10.1.5.4 of the ***HUB-VNG*** (Remember this local address BGP ip "might" be different in your lab).
	
- It can be seen that the ***HUB-VNG*** is learning the source prefix (10.0.0.0/16) through this AS Path: 65001 (***On-Prem-VNG***), 65002 (***CSR***), and 65515 (***Route Server***). It also learned the destination prefix (10.2.0.0/16) directly from ASN 65003 (***On-Prem1-VNG***).

![image](https://user-images.githubusercontent.com/78562461/140245775-a38fda90-bc11-447c-bd40-5ab0fb2d6a6c.png)


6. **Check On-Prem1-VNG:**
 
 Navigate to Virtual Network Gateways ->  On-Prem1-VNG -> Monitoring -> BGP Peers
	
👉 **BGP Peers:** 
	
- 10.1.5.4 is the BGP peer address of ***HUB-VNG*** gateway, 10.2.2.30 is the local BGP ip for the ***On-Prem1-VNG*** gateway. From below we see they are successfully connected.

![image](https://user-images.githubusercontent.com/78562461/140246508-75fcae89-2fc4-4ce4-ab97-a021fe1bb4f4.png)


👉 **Learned Routes:** 
	
- As we know now, 10.0.0.0/16 is the ***On-Prem-Vnet*** prefix which is where the source VM (***On-Prem-VM***) is located. Let us look at the AS Path which explain how this route has been learned: ASN 65001 refer to ***On-Prem-VNG*** gateway, it is the gateway originally advertised this route, then advertised it through eBGP peering over Ipsec to ***CSR*** NVA which has ASN 65002, and as the ***CSR*** has eBGP peering with ARS both instances it advertised this route to the ARS which has the ASN 65515. ARS has the
**Branch-to-Branch** feature enabled, so it advertised this route to the ***HUB-VNG*** gateway which has the ASN 65004, and the ***HUB-VNG*** advertised it down to the ***On-Prem1-VNG*** gateway through the eBGP peering over Ipsec.
	
💡 Without **Branch-to-Branch** feature enabled on ARS, 10.0.0.0/16 will not be advertised from ***CSR*** to the ***HUB-VNG***, and 10.2.0.0/16 will not be advertised from ***HUB-VNG*** gateway to the ***CSR***. 
	
- 10.1.0.0/16 is the ***HUB-SCUS*** Vnet prefix that is learned through eBGP peering over Ipsec from peer 10.1.5.4 (***HUB-VNG*** gateway BGP peer ip).

![image](https://user-images.githubusercontent.com/78562461/140253979-64f592e6-2b66-4001-8376-1c2091dee1e0.png)


7. **Lastly**, ***On-Prem1-VM***:

 Navigate to Network interfaces -> On-Prem1-VMNIC -> Help -> Effective routes:
	
- We can see that the VM learned 10.0.0.0/16 (where the source VM located), 10.1.5.4 is the BGP peer ip of the ***HUB-VNG*** gateway, 10.1.0.0/16 is the ***HUB-SCUS*** Vnet prefix, all those routes have been learned with Next Hop Type **virtual network gateway** which refer to ***On-Prem1-VNG*** gateway.

![image](https://user-images.githubusercontent.com/78562461/140254549-9f4af6a0-daca-47c6-bf8b-e997f3d0a82f.png)


🙂 From above we see that source ***On-Prem-VM*** (10.0.10.4) knows the route to destination ***On-Prem1-VM*** (10.2.10.4) and vice versa with no UDR has been used, and we see ping work fine:

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

In this scenario we will explore how a peered Vnet can learn routes from Azure Route Server that is deployed in peered Vnet, what will the peered ***Spoke-VM*** learn and what the next hop will be when having Virtual Network Gateway combined with ARS in same remote Vnet.

  
For this scenario we will add the following component:

1. ***Spoke-Vnet*** that will peer with the ***HUB-SCUS*** Vnet where the ARS is deployed. 

2. ***Spoke-VM*** to check on routing and test connectivity.

After deploying above the Network will look as below:


![image](https://user-images.githubusercontent.com/78562461/143797709-23374e81-1eef-41d5-a769-722f8462847e.png)


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
	
- 10.0.0.0/16 is the ***On-Prem-Vnet*** prefix that is learned from ***CSR*** NVA (10.1.1.4) through Azure Route Server as shown in the above scenarios. Note that next hop shows the ***CSR*** NVA IP and not the Azure Route Server local instances, as **ARS will not be in the data path, it only exchange the BGP routes with the NVA and with the Virtual Network Gateway when the Branch-to-Branch feature enabled.** 


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

🙂 To verify connectivity,ping ***HUB-VM*** (10.1.10.4), ping ***On-Prem-VM*** (10.0.10.4), and ***On-prem1-VM*** (10.2.10.4) from ***Spoke-VM*** (10.4.10.4), all pings will work fine: 

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

