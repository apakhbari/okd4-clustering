# OKD4 Clustering Setup
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


- Section1: Theoretical
  - Tips & Tricks
  - OpenShift Components
  - Minimum resource requirements
  - Producing an ignition config
  - Commands

- Section2: Setting up an OKD 4.5 Cluster

- Section3: Implementation
  - Schema
  - Cluster
  - okd4-services
  - okd4-pfsense
  - okd4-control-plane
  - okd4-compute
  - Cases
- Section4: acknowledgment


## 4: acknowledgment
### Contributors

APA 🖖🏻

### Links
- Installing a cluster on vSphere with user-provisioned infrastructure - OpenShift Documebtation: https://docs.openshift.com/container-platform/4.8/installing/installing_vsphere/installing-vsphere.html
---
- Install OpenShift 4 on Bare Metal - UPI - GitHub: https://github.com/ryanhay/ocp4-metal-install
- Guide: Installing an OKD 4.5 Cluster - Medium: https://medium.com/@craig_robinson/guide-installing-an-okd-4-5-cluster-508a2631cbee
- Guide: Installing an OKD 4.5 Cluster - Medium: [Guide: OKD 4.5 Single Node Cluster | by Craig Robinson | The Startup | Medium](https://medium.com/swlh/guide-okd-4-5-single-node-cluster-832693cb752b)
---
- OC monitoring: https://docs.openshift.com/container-platform/4.9/monitoring/monitoring-overview.html
---
- https://cloud.redhat.com/blog/provisioning-devops-on-openshift-using-helm-in-5-steps-from-zero-to-hero

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