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

### TO EDIT
- ALERT
- (HERE)
- vlan
- ip
- 32.20

## Table of contents:

1. Schema
2. Implementing prerequisites
3. Implementing OKD 4.5 Cluster
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

## 2- Implementing prerequisites
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

- Enable and start named:
```
sudo systemctl enable named
sudo systemctl start named
sudo systemctl status named
```

- Create firewall rules:
```
sudo firewall-cmd --permanent --add-port=53/udp
sudo firewall-cmd --reload
```

- Change the DNS on the okd4-service NIC that is attached to the VM Network (not OKD)
to 127.0.0.1.
- Restart the network services on the okd4-services VM:
```
sudo systemctl restart NetworkManager
```

- Test DNS on the okd4-services.
```
dig okd.local
dig –x 192.168.1.210
```

### Install HAProxy:
```
sudo dnf install haproxy -y
```

- Copy haproxy config from the git directory:
```
sudo cp haproxy.cfg /etc/haproxy/haproxy.cfg
```

- Start, enable, and verify HA Proxy service:
```
sudo setsebool -P haproxy_connect_any 1
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```

- Add OKD firewall ports:
```
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=22623/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

- Enable and Start httpd service/Allow port 8080 on the firewall:
```
sudo setsebool -P httpd_read_user_content 1
sudo systemctl enable httpd
sudo systemctl start httpd
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

- Test the webserver:
```
curl localhost:8080
```


### Install Apache/HTTPD
```
sudo dnf install -y httpd
```

- Change httpd to listen port to 8080:
```
sudo sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf
```

- Congratulations, You Are Half Way There!
- Congrats! You should now have a separate home lab environment setup and ready for ODK. Now we can start the install.


## 3- Implementing OKD 4.5 Cluster
### Download the openshift-installer and oc client:
- SSH to the okd4-services VM
- To download the latest oc client and openshift-install binaries, you need to use an
existing version of the oc client.
- Download the 4.5 version of the oc client and openshift-install from the OKD releases
page. Example:
```
wget https://github.com/openshift/okd/releases/download/4.5.0-0.okd-
2020-07-14-153706-ga/openshift-client-linux-4.5.0-0.okd-2020-07-14-
153706-ga.tar.gz
wget https://github.com/openshift/okd/releases/download/4.5.0-0.okd-
2020-07-14-153706-ga/openshift-install-linux-4.5.0-0.okd-2020-07-14-
153706-ga.tar.gz
```

- Extract the okd version of the oc client and openshift-install:
```
tar -zxvf openshift-client-linux-4.5.0-0.okd-2020-07-14-153706-ga.tar.gz
tar -zxvf openshift-install-linux-4.5.0-0.okd-2020-07-14-153706-ga.tar.gz
```

- Move the kubectl, oc, and openshift-install to /usr/local/bin and show the version:
```
sudo mv kubectl oc openshift-install /usr/local/bin/
oc version
openshift-install version
```

- The latest and recent releases are available at https://origin-release.svc.ci.openshift.org

### Setup the openshift-installer:
- In the install-config.yaml, you can either use a pull-secret from RedHat or the default of
“{“auths”:{“fake”:{“auth”: “bar”}}}” as the pull-secret.
- Generate an SSH key if you do not already have one.
```
ssh-keygen
```

- Create an install directory and copy the install-config.yaml file:
```
cd
mkdir install_dir
cp okd4_files/install-config.yaml ./install_dir
```

- Edit the install-config.yaml in the install_dir, insert your pull secret and ssh key, and
backup the install-config.yaml as it will be deleted in the next step:
- (HERE) ALERT: backup the install-config.yaml
```
vim ./install_dir/install-config.yaml
cp ./install_dir/install-config.yaml ./install_dir/install-config.yaml.bak
```

- Generate the Kubernetes manifests for the cluster, ignore the warning:
```
openshift-install create manifests --dir=install
_dir/
```

- Modify the cluster-scheduler-02-config.yaml manifest file to prevent Pods from being
scheduled on the control plane machines:
```
sed -i 's/mastersSchedulable: true/mastersSchedulable: False/'
install_dir/manifests/cluster-scheduler-02-config.yml
```

- Now you can create the ignition-configs:
```
openshift-install create ignition-configs --dir=install_dir/
```
- ALERT (HERE) Note: If you reuse the install_dir, make sure it is empty. Hidden files are created after
generating the configs, and they should be removed before you use the same folder on a
2nd attempt.

### Host ignition and Fedora CoreOS files on the webserver:
- Create okd4 directory in /var/www/html:
```
sudo mkdir /var/www/html/okd4
```

- Copy the install_dir contents to /var/www/html/okd4 and set permissions:
```
sudo cp -R install_dir/* /var/www/html/okd4/
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/
```

- Test the webserver:
```
curl localhost:8080/okd4/metadata.json
```

- Download the Fedora CoreOS bare-metal bios image and sig files and shorten the file
names:
```
cd /var/www/html/okd4/
sudo wget https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/32.20200629.3.0/x86_64/fedora-coreos-32.20200629.3.0-metal.x86_64.raw.xz
sudo wget https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/32.20200629.3.0/x86_64/fedora-coreos-32.20200629.3.0-metal.x86_64.raw.xz.sig

sudo mv fedora-coreos-32.20200629.3.0-metal.x86_64.raw.xz fcos.raw.xz.sig fcos.raw.xz
sudo mv fedora-coreos-32.20200629.3.0-metal.x86_64.raw.xz.sig fcos.raw.xz.sig fcos.raw.xz.sig
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/
```

### Starting the bootstrap node:

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