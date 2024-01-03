# OKD4-Clustering
```
   ____  _  _______  _  _           _____ _           _            _             
  / __ \| |/ /  __ \| || |         / ____| |         | |          (_)            
 | |  | | ' /| |  | | || |_ ______| |    | |_   _ ___| |_ ___ _ __ _ _ __   __ _ 
 | |  | |  < | |  | |__   _|______| |    | | | | / __| __/ _ \ '__| | '_ \ / _` |
 | |__| | . \| |__| |  | |        | |____| | |_| \__ \ ||  __/ |  | | | | | (_| |
  \____/|_|\_\_____/   |_|         \_____|_|\__,_|___/\__\___|_|  |_|_| |_|\__, |
                                                                            __/ |
                                                                           |___/ 
```
---

## Index:

---

- Section1: Theoretical
  - Tips & Tricks
  - OpenShift Components
  - Minimum resource requirements
  - Producing an ignition config
  - Commands
  - Cases


- Section2: Implementation
  - Schema
  - Cluster
  - okd4-services
  - okd4-pfsense
  - okd4-control-plane
  - okd4-compute
- Section3: References

---

# Section1: Theoretical

---

## Tips & Tricks

- All Components are running in Containers, e.g: scheduler, apiserver
- Core Features of OpenShift Cluster:
  - Fedora CoreOS: Immutable Container OS, main purpose of it is to run containers
  - CRI-O: Default container runtime
- RedHat Enterprise Linux CoreOS (released: 2019) is just for Openshift clusters while Fedora CoreOS (released: 2020) has mostly same features but is more suitable with general use cases.
- Immutability of CoreOS:
  - OStree: CoreOS has no package manager, it uses libostreee. Instead of changing your system package per package and file per file you create new file system layer. Every change to system also requires a reboot. --&gt; $ rpm-ostree
  - Ignition: Customize machine similar to how you customize it with configuration management tools.
- CoreOS encourages you to use all services In a containerized environment.
- Fedora CoreOS has some problems with SELinux updates, so if you make a change in SELinux policies ostree will stop updating policies with each Fedora version update --&gt; workaround: set temporary SELinux booleans on every boot instead of making it persistnet.

---

- its best practice to have a project for each application
- when running apps in OpenShift, logs of apps are being written in etcd database with json format
- Pod IP adresses are volatile, pods are not addressed directly, but services are used instead
- if you get an image from DockerHub which needs root privilege, you can bypass it running it with kubeadmin account
- if OpenShift is running in a private network, webhooks don’t work. To manually restart the build procedure, use $ oc start-build , to trigger the buildconfig again
- SELinux may be an issue: if host based storage is mounted in a container, use the following on the host folder to ensure SELinux is set correctly
  - $ semanage fcontext -a -t container_file_t/hostfolder(/.\*)?
  - $ restorecon -R /hostfolder
 
## OpenShift Components

OpenShift Resources : (defined in OpenShift APIs & stored in etcd database):

- pod : a running instance of an app, has IP, could have Volume
- ReplicaSet (formerly ReplicationController) : the resource that takes care of running multiple instances of pods
- Deployment (formerly DeploymentConfig) : resource that adds cluster properties to running the deployment, such as update strategy and replication. the ReplicaSet is managed by Deployment
- Service: used to load balance ingress traffic between the different Pod instances. (an API based Load Balancer)
- Route (OpenShift specific): used to expose a URL that provides access to services

APIs:

- for scalability we have replica set + deployment

- for accessability we have service + route

- for storing variables we have configMap + secret is for configuration and variables in a secure way

- for storage we have PV and storage can be dynamically created by storage class

Labels

- automatically or manually applied to workloads in OpenShift, used as a selector

- Replicasets use labels to monitor availability of pods, if labels change or deleted then replicaset is going to create new pods because availability of pods are being monitored using labels 

- Services use labels to connect to Pods with a matching label

- Admins can use labels to make filtering or scheduling Pods easier

pod to service connection

- services are using selector labels to find Pods they should connect to

- Pods themselves know which services they are connected to by two environment variables that are automatically assigned to running Pods

- SVC_NAME_SERVICE_HOST : services IP address

- SVC_NAME_SERVICE_PORT : services port

- services also automatically register with k8s internal DNS server, which makes them accessible through DNS as 

- SVC_NAME.PROJECT_NAME.svc.clustername

- cluster name can be obtained using $ oc config get-cluster , or $ oc config current-contex

- OpenShift runs a default router in the openshift-ingress namespace, the ROUTER_CANONICAL_HOSTNAME variable defines how this router is accessible from the outside
  - $ oc get pods -n openshift-ingress
  - $ oc describe pods -n openshift-ingress router-default-\[TAB\]

- A persistent Volume Claim (PVC) is used to request access to storage, applications are configured to use a specific PVC by referring to their name. A PVC has an access mode, and a resource request, but does not connect to a specific type of PV, that leaves the decision of what to connect to the cluster, to better determine what to connect to, StorageClass can be used as a matching label between the PV and the PVC. after requesting access to the PV, the PVC will show as bound

Template

- when an app is defined, typically a set of related resources using the same parameters is created

- Templates can be used to make creating these resources easier

- Resource attributes are defined as templates parameters

- Template parameters can be set statically, or generated dynamically

- OpenShift comes with a set of default templates; custom templates can be added as well

- $ oc get templates -n openshift —&gt; list default templates

- Each template contains specific sections

- objects : defines a list of resources that will be created

- parameters : defines parameters that are used in the template objects

- $ oc describe template templatename —&gt; list parameters that are used

- $ oc process —parameters templatename

- To generate an app from a template, first need to export the template

- $ oc get template mariadb-ephemeral -0 yaml -n openshift &gt; mariadb-ephemeral.yaml

- Next, you need to identify parameters that need to be set, and process the template with all of its parameters

- $ oc process —parameters mariadb-ephemeral -n openshift

- $ oc process -f mariadb-ephemeral.yaml -p MYSQL_USER=anna -p MYSQL_PASSWORD=password -p MYSQL_DATABASE=books | oc create -f -

- using oc new app with templates (most apps created like this must use —as-deployment-config in most cases, because most of them are still using deployment-config and not deployment)

- $ oc new-app —template=mariadb-ephemeral -p MYSQL_USER=anna -p MYSQL_PASSWORD=ann -p MYSQL_DATABASE=videos —as-deployment-config

ImageStream

- when running container-based apps, images need to be pulled
- These images are stored in the internal OpenShift image registry
- To access images in the image registry, an ImageStream resource is used, which identifies specific images using SHA ID
- ImageStreams store images using tags, which makes it easy to refer to different versions of images
- OpenShift images are not managed directly from the registry, but by using the ImageStream resource —&gt; DO NOT manage OpenShift images directly, only through ImageStream

- when a new app is created with $ oc create deploy or $ oc new-app, image registries will be contacted to check for a newer image, to prevent this use —image-stream to refer to an existing ImaegStream and use the image from the internall image registry. use $ oc new-app -L , to list all existing image streams and tags included. tgas can be combined with the —image-stream argument. for instance $ oc new-app —name=whatever —image-stream=nginx:1.18-ubi8

- OpenShift can do port forwarding as well, to expose a port on the computer where the oc client is used . It is OpenShift native and doesn’t need to stop Pod $ oc port-forwarding mydb 33060 3306

### Minimum resource requirements

Each cluster machine must meet the following minimum requirements:

| **Machine** | **Operating System** | **vCPU** | **Virtual RAM** | **Storage** |
| --- | --- | --- | --- | --- |
| Bootstrap | RHCOS | 4 | 16 GB | 120 GB |
| Control plane | RHCOS | 4 | 16 GB | 120 GB |
| Compute | RHCOS or RHEL 7.6 | 2 | 8 GB | 120 GB |

Refrence of table: <https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html?extIdCarryOver=true&sc_cid=701f2000001Css5AAC#minimum-resource-requirements_installing-vsphere>

## **Producing an Ignition Config**

### **Ignition overview**

- Ignition is a provisioning utility that reads a configuration file (in JSON format) and provisions a Fedora CoreOS system based on that configuration. Configurable components include storage and filesystems, systemd units, and users.
- Ignition runs only once during the first boot of the system (while in the initramfs). Because Ignition runs so early in the boot process, it can re-partition disks, format filesystems, create users, and write files before the userspace begins to boot. As a result, systemd services are already written to disk when systemd starts, speeding the time to boot.

### **Configuration process**

- Ignition configurations are formatted as JSON, which is quick and easy for a machine to read. However, these files are not easy for humans to read or write. The solution is a two-step configuration process that is friendly for both humans and machines:

  1\. Produce a YAML-formatted Butane config.

  2\. Run Butane to convert the YAML file into a JSON Ignition config.


- During the transpilation process, Butane verifies the syntax of the YAML file, which can catch errors before you use it to launch the FCOS system.
- Once you have an Ignition (`.ign`) file, you can use it to boot an FCOS system in a VM or install it on bare metal.\\

```
$ butane --pretty --strict config.yaml > config.ign
```

### Networking

<https://docs.fedoraproject.org/en-US/fedora-coreos/sysconfig-network-configuration/>

🟥 IMPORTANT: Do not forget to add --copy-network argument 🟥

```
$ sudo coreos-installer install /dev/sda --copy-network --ignition-url URL
```

1. Use $ nmtui
2. Add this to your yaml file:

```
variant: fcos
version: 1.4.0
storage:
  files:
    - path: /etc/NetworkManager/system-connections/ens2.nmconnection
      mode: 0600
      contents:
        inline: |
          [connection]
          id=ens2
          type=ethernet
          interface-name=ens2
          [ipv4]
          address1=10.10.10.10/24,10.10.10.1
          dns=8.8.8.8;
          dns-search=
          may-fail=false
          method=manual
```

Some Good Resources

<https://www.youtube.com/watch?v=2eEiVYelFTo&t=128s>

<https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/>

<https://coreos.github.io/zincati/usage/updates-strategy/>


## Commands

- $ oc api-resources —&gt; show a list of all resources and the specific API collection they are coming from. if its version is 1 or v1 it is coming from k8s if not it is added by OpenShift

- $ oc explain \[name of resource\] —&gt; get more information about API resource

- $ oc explain \[name of resource\].spec —&gt; information about API resources that could be inside yaml file

- $ oc projects , for developer or $ oc get ns , as adminstrator to get an overview of all projects

- $ oc new-app —&gt; primary tool for running apps, can build images from different sources:
  - Dockerfile
  - Image from any image repository
  - Directly from source code
  - Indirectly from sorce code, using Source to image (s2i)

- $ oc get all —&gt; find all application resources
- $ oc get all -A —&gt; an overview of resources in all projects

- $ oc create deploy mynginx —image=bitnami/nginx —dry-run=client -o yaml &gt; mynginx.yaml --&gt; see a deployment in order to edit and create later

- $ oc config get-cluster , or $ oc config current-contex --&gt; cluster name can be obtained

- $ oc expose service —&gt; generates a DNS name that looks like routname.projectname.defaultdomain , default domain is a wildcard DNS domain that is configured while installing OpenShift, and matches the OpenShift DNS name, on CRC the default domain is set to apps-crc.testing . The external DNS server needs to be configured with a wildcard DNS name that resolves to the load balancer that implements the route

- $ oc get routes --all-namespaces | grep -i console-openshift -- &gt;get all routes

- $ oc config set-context --current --namespace=&lt;namespace-name&gt; --&gt; change namespace

- oc project | cut -d '"' -f2 --&gt; Command to get current namespace

- oc scale deploy &lt;deployment_name&gt; -n &lt;namespace&gt; --replicas &lt;number_of_replicas&gt; --&gt; change number of replicas


## Cases
- Operator hub only showing community-operators --&gt; In Cluster's OperatorHub, changed disabled default sources to false
- Gracefully shutdown ckuster: <https://docs.openshift.com/container-platform/4.8/backup_and_restore/graceful-cluster-shutdown.html>

---

## Section2: Implementation

---

## Schema

Cluster consists of:

| Machine | OS | IP + MAC Address | Resources |
| --- | --- | --- | --- |
| <span style="color: rgb(229, 229, 229)">okd4-services (DNS/LB/WEB/NFS) \[Helper Node\]</span> | <span style="color: rgb(163, 163, 163)">Fedora Workstation 39</span> | eth1: LAN: 00:0c:29:8a:0b:32 - LAN: 192.168.1.210 / eth2: WAN: <span style="color: rgb(163, 163, 163)">00:0c:29:8a:0b:28 - WAN: 192.168.2.60</span> | CPU: 8 - Memory: 8 - HDD: 140 GB - Network: OKD(LAN) + VM(WAN) |
| <span style="color: rgb(229, 229, 229)">okd4-bootstrap (BootStrap Node)</span> | <span style="color: rgb(163, 163, 163)">ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz</span> | 00:0c:29:2a:73:13 - 192.168.1.200 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| <span style="color: rgb(229, 229, 229)">okd4-pfsense (FireWall - Router - DHCP)</span> | <span style="color: rgb(163, 163, 163)">FreeBSD</span> | eth1: LAN: <span style="color: rgb(163, 163, 163)">00:0c:29:2c:d5:43 - </span>LAN: 192.168.1.1 / eth2: WAN: <span style="color: rgb(163, 163, 163)">00:0c:29:2c:d5:39 - </span>WAN: 192.168.2.135 | CPU: 2 - Memory: 4 - HDD: 25 GB - Network: OKD(LAN) + VM(WAN) |
| <span style="color: rgb(229, 229, 229)">okd4-control-plane-1</span> | <span style="color: rgb(163, 163, 163)">ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz</span> | 00:0c:29:9a:77:53 - 192.168.1.201 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| <span style="color: rgb(229, 229, 229)">okd4-control-plane-2</span> | <span style="color: rgb(163, 163, 163)">ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz</span> | 00:0c:29:53:5d:97 - 192.168.1.202 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| <span style="color: rgb(229, 229, 229)">okd4-control-plane-3</span> | <span style="color: rgb(163, 163, 163)">ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz</span> | 00:0c:29:89:be:d5 - 192.168.1.203 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| <span style="color: rgb(229, 229, 229)">okd4-compute-1</span> | <span style="color: rgb(163, 163, 163)">ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz</span> | 00:0c:29:39:19:c1 - 192.168.1.204 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |
| <span style="color: rgb(229, 229, 229)">okd4-compute-2</span> | ISO: fedora-coreos-39.20231101.3.0-live.x86_64.iso - RAW: fedora-coreos-39.20231101.3.0-metal.x86_64.raw.xz | 00:0c:29:54:8c:a7 - 192.168.1.205 | CPU: 8 - Memory: 18 - HDD: 140 GB - Network: OKD |



## Cluster

- export enviormental variables to use bash: $ export KUBECONFIG=\~/install_dir/auth/kubeconfig

Persistent Volume Claim (PVC):

| NAMESPACE | NAME | STATUS | VOLUME | CAPACITY | ACCESS MODE | STORAGECLASS |
| --- | --- | --- | --- | --- | --- | --- |
| openshift-image-registry | image-registry-storage | Bound | registry-pv | 100Gi | RWX |  |
|  |  |  |  |  |  |  |
|  |  |  |  |  |  |  |

Persistent Volume (PV):

| NAME | CAPACITY | ACCESS MODE | RECLAIM POLICY | STATUS | CLAIM | STORAGECLASS | REASON |
| --- | --- | --- | --- | --- | --- | --- | --- |
| registry-pv | 100Gi | RWX | Retain | Bound | openshift-image-registry/image-registry-storage |  |  |
|  |  |  |  |  |  |  |  |
|  |  |  |  |  |  |  |  |

Routes:

$ oc get routes --all-namespaces | grep -i console-openshift

- console [console-openshift-console.apps.lab.qtroom.ir](http://console-openshift-console.apps.lab.qtroom.ir) console https reencrypt/Redirect None
- downloads [downloads-openshift-console.apps.lab.qtroom.ir](http://downloads-openshift-console.apps.lab.qtroom.ir) downloads http edge/Redirect None

NameSpaces:

- NFS

StorageClass:

- Default StorageClass : nfs-client

**Used this two resources:** <https://www.youtube.com/watch?v=6DmEp0kXUOI> **+ NFS Subdir External Provisioner:** <https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner>

IMPORTANT: When we delete a PV, it will not remain on nfs server

Additional helm repos:

- redhat-cop
- bitnami-helm-reopo
- 

**Using OpenShift’s Internal Registry:**

Accessing the OpenShift 4.x Internal Registry via an OpenShift “Route” : 

```
$ oc get route -n openshift-image-registry
```
The output should be of the form:
```
NAME            HOST/PORT
default-route   default-route-openshift-image-registry.apps-crc.testing
```
If the route is not exposed, the following command can be run:

```
$ oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```

We will dynamically create an environment variable with the name of route to the OpenShift registry for use in the remainder of this article. The route will have “/openshift” appended as this is a project that all users can access:
```
$ REGISTRY="$(oc get route/default-route -n openshift-image-registry -o=jsonpath='{.spec.host}')/openshift"
```

## okd4-services

packages that are installed on it:

- Google Chrome


- VIM


- GParted


- XRDP - TigerVnc
- named
- haproxy
- httpd
- jq 1.7
- nfs-utils + rpcbind
- Freedom of developer : <https://github.com/freedomofdevelopers/fod>
- OC CLI
- Tekton CLI

firewall:

- IPTables
- \- open ports:

  \- 3389/TCP \[RDP\]

  \- 53/UDP \[DNS\]

  \- 6443+22623 / TCP \[haproxy over http & https\]

  \- 8080/TCP \[apache httpd\]

  \- 111/TCP+UDP \[NFS: portmapper requests + SUN Remote Procedure Call (sunrpc) (rpcbind)\]

  \- 2049/TCP+UDP \[NFS Server Daemon (nfsd)\]

  \- 4045/TCP+UDP \[NFS lock daemon/manager (lockd)\]

  \- 4046/TCP \[NFS: for reboot detection and file lock recovery\]

  \- 4045:4047/TCP \[NFS: lockd, statd, mountd\]

  \- 1110/TCP \[NFS: Cluster status info (nfsd-status)\]

  \- 1110/UDP \[NFS: Client status info (nfsd-keepalive)\]

IPTables rules:

- \-A INPUT -p tcp -m tcp --dport 8080 -j ACCEPT

  \-A INPUT -p tcp -m tcp --dport 22623 -j ACCEPT

  \-A INPUT -p tcp -m tcp --dport 6443 -j ACCEPT

  \-A INPUT -p udp -m udp --dport 53 -j ACCEPT

  \-A INPUT -p tcp -m tcp --dport 3389 -j ACCEPT

  \-A INPUT -p tcp --dport 111 -j ACCEPT

  \-A INPUT -p tcp --sport 2049 -j ACCEPT

  \-A INPUT -p tcp --dport 4046 -j ACCEPT

  \-A INPUT -p udp -m udp --dport 4045 -j ACCEPT

  \-A INPUT -p tcp -m tcp --dport 4045 -j ACCEPT

  \-A INPUT -p udp -m udp --dport 2049 -j ACCEPT

  \-A INPUT -p udp -m udp --dport 111 -j ACCEPT

  \-A INPUT -p udp -m udp --dport 1110 -j ACCEPT

  \-A INPUT -p tcp -m tcp --dport 1110 -j ACCEPT

  \-A OUTPUT -p tcp -m tcp --sport 8080 -j ACCEPT

  \-A OUTPUT -p tcp -m tcp --sport 22623 -j ACCEPT

  \-A OUTPUT -p tcp -m tcp --sport 6443 -j ACCEPT

  \-A OUTPUT -p udp -m udp --sport 53 -j ACCEPT

  \-A OUTPUT -p tcp -m tcp --sport 3389 -j ACCEPT

  \-A OUTPUT -p tcp --dport 111 -j ACCEPT

  \-A OUTPUT -p tcp --dport 2049 -j ACCEPT

  \-A OUTPUT -p tcp --dport 4045:4047 -j ACCEPT

  \-A OUTPUT -p udp -m udp --sport 4045 -j ACCEPT

  \-A OUTPUT -p tcp -m tcp --sport 4045 -j ACCEPT

  \-A OUTPUT -p udp -m udp --sport 2049 -j ACCEPT

  \-A OUTPUT -p udp -m udp --sport 111 -j ACCEPT

  \-A OUTPUT -p udp -m udp --sport 1110 -j ACCEPT

  \-A OUTPUT -p tcp -m tcp --sport 1110 -j ACCEPT

SELinux: Permissive

\
$ sudo setsebool -P haproxy_connect_any 1

$ sudo setsebool -P nfs_export_all_rw 1

named (DNS):

SSH

ssh is enabeld

RSA 3072:

3072 SHA256:Z9hzOmcdMa6HrYfYFu0rvSvhCKLw4vGXm0jQR94yXg4 adak@okd4-services (RSA)

SSH key Random Art:

```
+---[RSA 3072]----+
|                 |
|                 |
|      .       o  |
|   . o . o   . o |
|  . . E S = ..o  |
|  .. o.*.o +o=.. |
|  .o...o..o==*+  |
|  .+o.o.  o+Bo+  |
| ...o.o.   ..+++ |
+----[SHA256]-----+
```

Public Key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCqnlgiWgwZPrjj9pM7+6OGq3gUecTGIoryBDiriKHfbJHmG3pO7rFIogqqrumjsLkduTcP3kHFRQ3+biujiV+f8zdzaKiTf7IWtkIr4Ut4lOMC1WHANnZM2hIoCwROYikYVdCB/igKTCQSzkCDS/zGP6jmcuXm7tfC3WgH5pQ+Or8daTWXP09DyEAggmc2hKXEWPrxycAVSIWFx0VJTV7rBSBRNz1MXiwtvQzLolclpitEROuHnweQFrT0CIF5MxfDkLu369pCXW8be57jkW/RTsrDYMNYuaXuVncR8oWdPUm/awc/k5ptdvdjzCEZJH23Kka/Sll7bKiXxL8xMW9GehOWelXTb4ydzqFG+E/tRdvCJPz/mkP/EtlAdzWgFV8DqSvjLEpe28G/Qxe+A67Yf0/yiHUs1VjNdbfx+OX2K3yDREF9gpsKN86NRAdRDelsq0Rj4ZbRcya8FEEAZ549a1x9/AsEFYsF9RXrhEgKNYQZDyQwV2VTmgoS11aRdfk= adak@okd4-services

ED25519:

256 SHA256:f82G5Mf0eC+jY/41k++RDfarmpgcfgxcon1ClOmC7WY adak@okd4-services (ED25519)

SSH key Random Art:

```
+--[ED25519 256]--+
|          o      |
|         +       |
|      o o        |
|     . o + .     |
|      . S o . +  |
|       E * + B =+|
|      o  .* + B**|
|        o +++oooB|
|         =.=+=o=+|
+----[SHA256]-----+
```

Public Key: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAIiShK98JDdH8gPH8ZzXZ49ycz+uap+t515DvKFzMpf adak@okd4-services

NFS Export:

- /var/nfsshare 192.168.1.0/24(rw,sync,no_root_squash,no_all_squash,no_wdelay)
- /var/nfs-storage-class 192.168.1.0/24(rw,sync,no_root_squash,no_all_squash,no_wdelay)

## okd4-pfsense

Domain: qtroom.ir

Addresses:

- WAN: 192.168.2.135
- LAN: 192.168.1.1

ssh is enabeld

Services &gt; WOL (Wake On LAN) : Added Wake on Lan for bootstrap, compute-nodes & worker-nodes

## okd4-control-plane

port: 6443-22623 is used for haproxy

## okd4-compute

port: 6443-22623 is used for haproxy

Section3: References

---

# acknowledgment
## Contributors

APA 🖖🏻

## Links
Installing a cluster on vSphere with user-provisioned infrastructure - OpenShift Documebtation: <https://docs.openshift.com/container-platform/4.8/installing/installing_vsphere/installing-vsphere.html>
---

\
<span style="color: rgb(201, 209, 217)">Install OpenShift 4 on Bare Metal - UPI - GitHub: </span><https://github.com/ryanhay/ocp4-metal-install>

Guide: Installing an OKD 4.5 Cluster - Medium: <https://medium.com/@craig_robinson/guide-installing-an-okd-4-5-cluster-508a2631cbee>

Guide: Installing an OKD 4.5 Cluster - Medium: [Guide: OKD 4.5 Single Node Cluster | by Craig Robinson | The Startup | Medium](https://medium.com/swlh/guide-okd-4-5-single-node-cluster-832693cb752b)

---

OC monitoring: <https://docs.openshift.com/container-platform/4.9/monitoring/monitoring-overview.html>

---

<https://cloud.redhat.com/blog/provisioning-devops-on-openshift-using-helm-in-5-steps-from-zero-to-hero>


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
