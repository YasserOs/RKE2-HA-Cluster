# HA k8s cluster with RKE2
## Architecture

The architecture of RKE2 is split into Servers and Agents , where Servers represent the master nodes and Agents represent the worker nodes .
[Image]

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

rke2 (agent and server ) scripts can be configured through 2 ways , CLO flags and config files , we are going to take the 2nd approach and create a simple minimal config file in the default location that rke2 looks for ( we can provide a different file path through the cli flags )

``` bash
touch /etc/rancher/rke/config.yaml 
```
There are many options that can be configured for the server , you can refer to the rke2 documentation for the server config options https://docs.rke2.io/reference/server_config

The most important options that we are going to use are :
- server 
This is the ip of the server that all the other nodes will need to join the cluster , it can be the ip of any master node we created or a load balancer that sits in front of the master nodes 

- token 
This is the token that the cluster nodes will use to join the cluster , generated from initializing the 1st master node or we can provide our token 

- tls-san:
    - "xxx.xxx.xxx.xxx"

This option is used to include any ips/domain names in the server certificates of the api-servers of the master nodes , needed if we are going to use a load balancer to access the cluster's multiple api-servers (this loadbalancer ip is used in the kubeconfig file that kubectl uses to communicate with the cluster )

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
tls-san:
  - "20.4.3.208"  

save and exit file

systemctl enable rke2-server.service 
systemctl start rke2-server.service

now we created our 3 master nodes , we need to ssh into the 2 worker nodes and do a very similar installation 
============
ssh into worker 1 & 2 :

Download and install the rke2-agent script
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=agent sh -  

sudo mkdir -p /etc/rancher/rke2/ 

# change the ip to reflect your rancher1 ip
echo "server: https://<ip-1st-maser-node>:9345" > /etc/rancher/rke2/config.yaml

# change the Token to the one from rancher1 /var/lib/rancher/rke2/server/node-token 
echo "token: $TOKEN" >> /etc/rancher/rke2/config.yaml

# enable and start
systemctl enable rke2-agent.service
systemctl start rke2-agent.service

now we successfully installed a HA 5 nodes cluser and we can check the results by running kubectl get nodes on any machine with the kubeconfig file we copied earlier from /etc/rancher/rke2/rke2.yaml on any of the master nodes and changing the server ip to the loadbalancer ip


============================

Install a specific version :
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_VERSION=v1.24.10+rke2r1 sh -
Server:
sudo systemctl restart rke2-server

Agent:
sudo systemctl restart rke2-agent

=================================

Install rancher UI :
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=<LOAD_BALANCER_DomainName> \
  --set bootstrapPassword=admin

===============================
load balancer for rancher ui config :
add this to /etc/nginx/nginx.conf 

worker_processes 4;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

stream {
  upstream rancher_servers_http {
      least_conn;
      server 192.168.0.4:80 max_fails=3 fail_timeout=5s;
      server 192.168.0.9:80 max_fails=3 fail_timeout=5s;
      server 192.168.0.10:80 max_fails=3 fail_timeout=5s;
      server 192.168.0.7:80 max_fails=3 fail_timeout=5s;
      server 192.168.0.8:80 max_fails=3 fail_timeout=5s;
  }
  server {
      listen 80;
      proxy_pass rancher_servers_http;
  }

  upstream rancher_servers_https {
      least_conn;
      server 192.168.0.4:443 max_fails=3 fail_timeout=5s;
      server 192.168.0.9:443 max_fails=3 fail_timeout=5s;
      server 192.168.0.10:443 max_fails=3 fail_timeout=5s;
      server 192.168.0.7:443 max_fails=3 fail_timeout=5s;
      server 192.168.0.8:443 max_fails=3 fail_timeout=5s;
  }
  server {
      listen     443;
      proxy_pass rancher_servers_https;
  }

}