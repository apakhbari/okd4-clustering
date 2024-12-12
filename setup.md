# OKD4.5 Clustering Setup
```
   ____  _  _______  _  _           _____ _           _            _             
  / __ \| |/ /  __ \| || |         / ____| |         | |          (_)            
 | |  | | ' /| |  | | || |_ ______| |    | |_   _ ___| |_ ___ _ __ _ _ __   __ _ 
 | |  | | |  | |  | |__   _|______| |    | | | | / __| __/ _ \ '__| | '_ \ / _` |
 | |__| | . \| |__| |  | |        | |____| | |_| \__ \ ||  __/ |  | | | | | (_| |
  \____/|_|\_\_____/   |_|         \_____|_|\__,_|___/\__\___|_|  |_|_| |_|\__, |
                                                                            __/ |
                                                                           |___/ 
```

## Table of contents:

1. Schema
2. Implementation
3.  
4. Acknowledgment

## 1- Schema

### VM Schema

| Machine | OS | IP + MAC Address | Resources |
| --- | --- | --- | --- |
| okd4-services (DNS/LB/WEB/NFS) \[Helper Node\] | Fedora Workstation 39 | eth1: LAN: 00:0c:29:8a:0b:32 - LAN: 192.168.1.210 / eth2: WAN: 00:0c:29:8a:0b:28 - WAN: 192.168.2.60 | CPU: 8 - Memory: 8 - HDD: 140 GB - Network: OKD(LAN) + VM(WAN) |
| okd4-bootstrap (BootStrap Node) | ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz | 00:0c:29:2a:73:13 - 192.168.1.200 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| okd4-pfsense (FireWall - Router - DHCP) | FreeBSD | eth1: LAN: 00:0c:29:2c:d5:43 - LAN: 192.168.1.1 / eth2: WAN: 00:0c:29:2c:d5:39 - WAN: 192.168.2.135 | CPU: 2 - Memory: 4 - HDD: 25 GB - Network: OKD(LAN) + VM(WAN) |
| okd4-control-plane-1 | ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz | 00:0c:29:9a:77:53 - 192.168.1.201 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| okd4-control-plane-2 | ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz | 00:0c:29:53:5d:97 - 192.168.1.202 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| okd4-control-plane-3 | ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz | 00:0c:29:89:be:d5 - 192.168.1.203 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| okd4-compute-1 | ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz | 00:0c:29:39:19:c1 - 192.168.1.204 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| okd4-compute-2 | ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz | 00:0c:29:54:8c:a7 - 192.168.1.205 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |


### Network Schema
(HERE)

## 2- Implementation
- NOTE: In this guide, I used VMWare ESXi as the hypervisor (HERE) (USE ALERT md)
### Create a new network in VMWare for OKD
- Login to your VMWare Host. Select Networking → Port Groups → Add port group. Setup an OKD network on an unused VLAN, in my instance, VLAN 20.
- Name your Group and set your VLAN ID.

### Create the okd4-services VM:
- Download the Fedora Workstation 39 and upload it to your ESXi host datastore.
- Create a new Virtual Machine. Choose Guest OS as Linux and Select Fedora Workstation 39 (64-bit).
- Select the Datastore and customize the settings to 4 vCPU, 4GB RAM, 100GB HD. Add a
2nd network adapter to the OKD network. Attached the CentOS ISO. (HERE)
- Review your settings and click Finish.

### Install Fedora Workstation 39 on the okd4-services VM:
- Run through the CentOS 8 installation.
- For Software Selection, use Server with GUI and add the Guest Agents.
- Enable the NIC connected to the VM Network and set the hostname as okd4-services,
then click Apply and Done.
- Click “Begin Installation” to start the install.
- Set the Root password, and create an admin user.
- After the installation has completed, login, and update the OS.
``` (HERE)
sudo dnf install -y epel-release
sudo dnf update -y
sudo systemctl restart
```

- Setup XRDP for Remote Access from Home Network
``` (HERE)
sudo dnf install -y xrdp tigervnc-server
sudo systemctl enable --now xrxp
sudo firewall-cmd --zone=public --permanent --add-port=3389/tcp
sudo firewall-cmd --reload
```

- Download Google Chrome rpm and install along with git
``` (HERE)
sudo dnf install -y ~/Downloads/google-chrome-stable_current_x86_64.rpm git
```

### Create the okd4-pfsense VM:
- Download the pfSense ISO and upload it to your ESXi host’s datastore.
- Create a new Virtual Machine. Choose Guest OS as Other and Select FreeBSD 64-bit.
- Use the default template settings for resources.
- Select your home network for Network Adapter 1, and add a new network adapter using the OKD network.

### Setup the okd4-pfsense VM:
- Power on your pfSense VM and run through the installation using all the default values.
- Login to pfSense via your web-browser on the okd4-services VM. The default username is “admin” and the password is “pfsense”.
- After logging in, click next and use “okd4-pfsense” for hostname and “okd.local” for the domain and add 192.168.1.210 as the Primary DNS server.
- Use Defaults for WAN Configuration. Uncheck “Block RFC1918 Private Networks” since your home network is the “WAN” in this setup. Next.
- Use the default LAN IP and subnet mask. Set an admin password on the next screen.

### Create bootstrap, master, and worker nodes:
- Download the Fedora CoreOS Bare Metal ISO and upload it to your ESXi datastore.
- The latest stable version at the time of writing is fedora-coreos-39.20231101.3.0 (HERE) (ALERT md)
- Create the six ODK nodes (bootstrap, master, worker) on your ESXi host.

### Setup DHCP reservations:
- Compile a list of the OKD nodes MAC addresses by viewing the hardware configuration
of your VMs.
- Login into pfSense. Go to Services → DHCP Server and change your ending range IP to
192.168.1.99, and set the primary DNS server as 192.168.1.210, then click Save.
- On the DHCP Server, page click Add at the bottom.
- Fill in the MAC Address, IP Address, and Hostname, then click save. Do this for each
ODK VM. Also, include the okd4-services MAC on the OKD network while you are at it.
Click Apply Changes at the top of the page when complete.

### Configure okd4-services VM to host various services:
- The okd4-services VM is used to provide DNS, NFS exports, web server, and load
balancing.
- Copy the MAC address on the VM Hardware configuration page for the NIC connected to
the OKD network and set up a DHCP Reservation for this VM using the IP address
192.168.1.210.
- Hit “Apply Changes” at the top of the DHCP page when completed.
- Open a terminal on the okd4-services VM and clone the clustering repo that contains
the DNS, HAProxy, and install-conf.yaml example files:
``` (HERE)
cd
git clone https://github.com/apakhbari/okd4-clustering.git
cd okd4_files
```

### Install bind (DNS)
```
sudo dnf -y install bind bind-utils
```

- Copy the named config files and zones:
```
sudo cp named.conf /etc/named.conf
sudo cp named.conf.local /etc/named/
sudo mkdir /etc/named/zones
sudo cp db* /etc/named/zones
```






## 4- Acknowledgment
### Contributors

APA 🖖🏻

### Links

### © APA, 2022-2024

```                                                                                                       
  aaaaaaaaaaaaa  ppppp   ppppppppp     aaaaaaaaaaaaa   
  a::::::::::::a p::::ppp:::::::::p    a::::::::::::a  
  aaaaaaaaa:::::ap:::::::::::::::::p   aaaaaaaaa:::::a 
           a::::app::::::ppppp::::::p           a::::a 
    aaaaaaa:::::a p:::::p     p:::::p    aaaaaaa:::::a 
  aa::::::::::::a p:::::p     p:::::p  aa::::::::::::a 
 a::::aaaa::::::a p:::::p     p:::::p a::::aaaa::::::a 
a::::a    a:::::a p:::::p    p::::::pa::::a    a:::::a 
a::::a    a:::::a p:::::ppppp:::::::pa::::a    a:::::a 
a:::::aaaa::::::a p::::::::::::::::p a:::::aaaa::::::a 
 a::::::::::aa:::ap::::::::::::::pp   a::::::::::aa:::a
  aaaaaaaaaa  aaaap::::::pppppppp      aaaaaaaaaa  aaaa
                  p:::::p                              
                  p:::::p                              
                 p:::::::p                             
                 p:::::::p                             
                 p:::::::p                             
                 ppppppppp                             
                                                       
```