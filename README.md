# HA k8s cluster with RKE2
## Architecture

The architecture of RKE2 is split into Servers and Agents , where Servers represent the master nodes and Agents represent the worker nodes .

![App Screenshot](https://docs.rke2.io/assets/images/overview-783b5a0a7e319dc96a2db8473dc83f3d.png)

## Requirements
There are 2 main scripts that we need to install the cluster , RKE2 server script which we are going to run on the nodes we want to assign as masters , and RKE2 agent script on nodes assigned to be workers
## Installed Components 
### All Nodes :
- Kubelet ( systemd service )
- containerd ( container runtime )
### Server :
- Controlplane components ( static pods )
    - api-server
    - controller-manager
    - scheduler
    - etcd
- Helm Controller
### Agent :
- Helm Deployed Charts 
    - Canal CNI ( default cni )
    - Nginx ingress controller
    - Metrics server
    - Coredns
    - kube-proxy
---
## Installation
To deploy a HA cluster consisting of 5 nodes ( 3 masters and 2 workers ) follow these steps .

### General setup on All nodes :
``` bash
disable the firewall and swap
sudo systemctl stop ufw
sudo systemctl disable ufw
swapoff -a
```
### 1st Master Node
ssh into the 1st desired master node :

Download and install the rke2-server script
``` bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh -
```
``` bash
mkdir -p /etc/rancher/rke2
```

rke2 (agent and server ) scripts can be configured through 2 ways , CLI flags and config files , we are going to take the 2nd approach and create a simple minimal config file in the default location that rke2 looks for ( we can provide a different file path through the cli flags )

``` bash
touch /etc/rancher/rke/config.yaml 
```
There are many options that can be configured for the server , you can refer to the rke2 documentation for the server config options https://docs.rke2.io/reference/server_config

The most important options that we are going to use are :
- server : This is the ip of the server that all the other nodes will need to join the cluster , it can be the ip of any master node we created or a load balancer that sits in front of the master nodes 

- token : This is the token that the cluster nodes will use to join the cluster , generated from initializing the 1st master node or we can provide our token 

- tls-san:
    - "xxx.xxx.xxx.xxx"
\n This option is used to include any ips/domain names in the server certificates of the api-servers of the master nodes , needed if we are going to use a load balancer to access the cluster's multiple api-servers (this loadbalancer ip is used in the kubeconfig file that kubectl uses to communicate with the cluster )



---
### Note
Since this is the 1st master node to be initialized we won't need the server or the token options , we are going to take these values from the node after its initialized and use it in other nodes joining the cluster 

``` bash
nano /etc/rancher/rke2/config.yaml
```
### write this in the file ( this is the load balancer ip)
``` bash
tls-san:
    - "20.4.3.208"  
```
<img heught=100 width="500" alt="tls-san" src="https://user-images.githubusercontent.com/95745669/219389155-fa8d44fb-bce8-4141-a86b-cfa0a623d5ba.png">


save file and exit

### Start and enable the server for restarts - 
``` bash
systemctl enable rke2-server.service 
systemctl start rke2-server.service
```

now the server will pull the rke2 runtime image and will extracts the binaries from it , install ```kubelet``` and ```containerd``` as systemd services , create manifest files for the control plane components and deplpoy them as static pods , it will generate the token needed by other nodes to join the cluster in the path
```/var/lib/rancher/rke2/server/node-token``` 
 and will generate the ```kubeconfig``` file in the path ```/etc/rancher/rke2/rke2.yaml``` (we can copy this file and place it in any machine that has kubectl tool in ```~/.kube/config``` and manage the cluster from that machine , we only need to change the ip of the server that kubectl communicates with to be the loadblancer ip)


```bash
cat /var/lib/rancher/rke2/server/node-token
```
<img height=100 width="611" alt="token" src="https://user-images.githubusercontent.com/95745669/219389738-e523b8df-b372-40bc-a67c-34ec985dd6ee.png">

copy the token we will use it in the config file of other nodes joining the cluster

## 2nd & 3rd Master Nodes
ssh into Master nodes 2 and 3 :

download and install the same server script 
```bash 
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh -
```

```bash 
mkdir -p /etc/rancher/rke2 
```

here we are going to add the 2 other important options in the config file which are the server and token 

### change the ip to reflect your 1st Master Node ip
```bash
echo "server: https://<ip-1st-masterr-node>:9345" > /etc/rancher/rke2/config.yaml 
```

### change the Token to the one from 1st Master Node /var/lib/rancher/rke2/server/node-token 

```bash
echo "token: <copied-token>" >> /etc/rancher/rke2/config.yaml
```

```bash nano /etc/rancher/rke2/config```

add this option too in the file
```bash
tls-san:
  - "20.4.3.208"  
```
<img height= 100 width="750" alt="config-file" src="https://user-images.githubusercontent.com/95745669/219386918-333ed689-3481-4407-a0aa-9ed878bc6b7c.png">

save and exit file
```bash
systemctl enable rke2-server.service 
systemctl start rke2-server.service
```
## Now we finished installing the 3 server nodes as masters , we will continue with the workers ( agents )
---
### Worker Node 1 & 2 

ssh into worker 1 & 2 :

Download and install the rke2-agent script
```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=agent sh -  
```
```bash
sudo mkdir -p /etc/rancher/rke2/ 
```
### change the ip to reflect your 1st master node ip or any of the other masters ips
echo "server: https://<ip-1st-maser-node>:9345" > /etc/rancher/rke2/config.yaml

### change the Token to the one from the 1st master node /var/lib/rancher/rke2/server/node-token 
echo "token:<master-token> >> /etc/rancher/rke2/config.yaml
<img height=100 width="637" alt="agent-config" src="https://user-images.githubusercontent.com/95745669/219389907-fb3fbd26-34ce-41cd-bbaa-e12426f6e57f.png">

### enable and start agent
systemctl enable rke2-agent.service
systemctl start rke2-agent.service

now we successfully installed a HA 5 nodes cluser and we can check the results by running kubectl get nodes on any machine with the kubeconfig file we copied earlier from /etc/rancher/rke2/rke2.yaml on any of the master nodes and changing the server ip to the loadbalancer ip


---
## Cluster Upgrade or Downgrade 

we can downgrade or upgrade the cluster in a very easy and straiht forward manner , it's done by downloading and installing a specific rke2 cluster version from the [releases](https://github.com/rancher/rke2/releases) page , on every node of the cluster and restarting the service on the nodes one by one ( 1 master at a time , then 1 agent at a time ) 

Example we want our cluster version to be 1.24.10
```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_VERSION=v1.24.10+rke2r1 sh -
```
we will restart the service on our server (master) nodes one by one

Server:
sudo systemctl restart rke2-server

then the agents one by one
Agent:
sudo systemctl restart rke2-agent

### after all nodes services have been restarted the cluster will be ready with the required version
---
# Installing Rancher UI on the cluster

Rancher Installation is straight forward using their [Documentation](https://ranchermanager.docs.rancher.com/v2.6/pages-for-subheaders/install-upgrade-on-a-kubernetes-cluster)

it's installed using helm 
### Note
you can add a domain name for you loadbalancer in your /etc/hosts file if you aren't going to create a dns record on the internet
```bash
echo <Load-balancer-ip> <your-desired-domain-name> >> /etc/hosts
```
then use that domain name in this command 
```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=<LOAD_BALANCER_DomainName> \
  --set bootstrapPassword=admin
```

---
# Using Nginx as Proxy/Loadbalancer for Rancher UI & Kubectl 
we need to provision a linux virtual machine with a public ip and instll nginx on it and configure it as a loadbalancer for both kubectl commands & the rancher UI 

after creating the vm and ssh into it :
```bash 
sudo apt install nginx
```
### Loadbalancer config for kubectl commands :
```bash
mkdir -p /etc/nginx/tcpconf.d
```

```bash 
nano k8s.conf
```

copy this into the file 
```bash
upstream kubernetes {
server 192.168.0.7:6443;
server 192.168.0.8:6443;
server 192.168.0.10:6443;
}


server {
listen 6443;
proxy_pass kubernetes;
}
```


### load balancer for rancher ui config :
add this to /etc/nginx/nginx.conf 
```bash
load_module /usr/lib/nginx/modules/ngx_stream_module.so;
worker_processes 4;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

stream {
  upstream rancher_servers_http {
      least_conn;
      server <ip-of node where rancher is deployed on>:80 max_fails=3 fail_timeout=5s;
      server <ip-of node where rancher is deployed on>:80 max_fails=3 fail_timeout=5s;
      server <ip-of node where rancher is deployed on>:80 max_fails=3 fail_timeout=5s;

  }
  server {
      listen 80;
      proxy_pass rancher_servers_http;
  }

  upstream rancher_servers_https {
      least_conn;
      server <ip-of node where rancher is deployed on>:443 max_fails=3 fail_timeout=5s;
      server <ip-of node where rancher is deployed on>:443 max_fails=3 fail_timeout=5s;
      server <ip-of node where rancher is deployed on>:443 max_fails=3 fail_timeout=5s;
  server {
      listen     443;
      proxy_pass rancher_servers_https;
  }
  include /etc/nginx/tcpconf.d/*;
}
```

### Now we have HA 5 nodes cluster with loadbalancing enabled for kuebetl commands and rancher UI panel

