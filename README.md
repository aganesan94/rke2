# rke2
<!-- TOC -->
* [rke2](#rke2)
  * [What is RKE2?](#what-is-rke2)
  * [RKE2 - Set up and install (1 Control Plane, 2 Worker Nodes)](#rke2---set-up-and-install-1-control-plane-2-worker-nodes)
    * [Pre-requisites](#pre-requisites)
    * [Set up the rke2 control plane (also called server) on Node -1](#set-up-the-rke2-control-plane-also-called-server-on-node--1)
    * [Set up the rke2 worker (also called agent) on Node-2 and Node-3](#set-up-the-rke2-worker-also-called-agent-on-node-2-and-node-3)
  * [Rancher](#rancher)
    * [Pre-requisite](#pre-requisite)
    * [Install rancher](#install-rancher)
    * [Accessing Rancher via the UI (Use your hostname)](#accessing-rancher-via-the-ui-use-your-hostname)
  * [Longhorn](#longhorn)
    * [Install Longhorn](#install-longhorn)
    * [Go back to the host being set up](#go-back-to-the-host-being-set-up)
  * [References](#references)
<!-- TOC -->


## What is RKE2?

* RKE2 - Security focused Kubernetes - https://docs.rke2.io/
* Rancher - Multi-Cluster Kubernetes Management via UI - https://ranchermanager.docs.rancher.com/
* Longhorn - Unified storage layer - https://longhorn.io/
* Air Grapping:  An air-gapped network is akin to an island, safe, secure, and isolated from other networks that have lesser security and more significant threats.
* Neuvector Federation
* Hauler

## RKE2 - Set up and install (1 Control Plane, 2 Worker Nodes)

### Pre-requisites

Create 3 nodes on service provider of your choice with the following spec with ubuntu-2204

| name | Hostname        | ip              | memory | core | disk | os                            |
|------|-----------------|-----------------|--------|------|------|-------------------------------|
| rke1 | 143.244.139.181 | 143.244.139.181 | 8192   | 4    | 160  | 22.04.4 LTS (Jammy Jellyfish) |
| rke2 | 143.244.139.181 | 143.244.136.174 | 8192   | 4    | 160  | 22.04.4 LTS (Jammy Jellyfish) |
| rke3 | 143.244.139.181 | 143.244.139.182 | 8192   | 4    | 160  | 22.04.4 LTS (Jammy Jellyfish) |

```shell
# get updates, install nfs, and apply
sudo apt update && sudo apt upgrade -y

# Disable ufw
sudo systemctl disable --now ufw

# 
sudo apt install nfs-common net-tools -y  

# clean up
sudo apt autoremove -y
```

### Set up the rke2 control plane (also called server) on Node -1

```shell
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.26 INSTALL_RKE2_TYPE=server sh -
 
 # start and enable for restarts - 
systemctl enable --now rke2-server.service

# Verify if rke2 service is active
systemctl status rke2-server.service

# Ensure to have symlinks so kubectl commands can be enabled
ln -s $(find /var/lib/rancher/rke2/data/ -name kubectl) /usr/local/bin/kubectl

# add kubectl conf or add to path
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml 

# Verify the node
root@ubuntu-rke2-1:~# kubectl get node
NAME            STATUS   ROLES                       AGE   VERSION
ubuntu-rke2-1   Ready    control-plane,etcd,master   17m   v1.26.14+rke2r1

# obtain the token from rancher
# save this for rancher2 and rancher3
cat /var/lib/rancher/rke2/server/node-token

#The goal of the token process is to setup a control plane Mutual TLS (mtls) certificate termination.
root@ubuntu-rke2-1:~# cat /var/lib/rancher/rke2/server/node-token
K1073237ec96f322c98ec4ca766bc2290d4ebe56ec096e8834ee4afece268d86708::server:123f3ba5b4ca43c4f432b1a562624136
 ```

### Set up the rke2 worker (also called agent) on Node-2 and Node-3

```shell
# change this!
export RANCHER1_IP=143.244.139.181  

# and we can export the token from rke1
export TOKEN=K1073237ec96f322c98ec4ca766bc2290d4ebe56ec096e8834ee4afece268d86708::server:123f3ba5b4ca43c4f432b1a562624136

# we add INSTALL_RKE2_TYPE=agent
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.26 INSTALL_RKE2_TYPE=agent sh -  

# create config file
mkdir -p /etc/rancher/rke2/ 

# change the ip to reflect your rancher1 ip
echo "server: https://$RANCHER1_IP:9345" > /etc/rancher/rke2/config.yaml

# change the Token to the one from rancher1 /var/lib/rancher/rke2/server/node-token 
echo "token: $TOKEN" >> /etc/rancher/rke2/config.yaml

# enable and start
systemctl enable --now rke2-agent.service

# check status
systemctl status rke2-agent.service

# Set up a symlink for the kubectl version
ln -s $(find /var/lib/rancher/rke2/data/ -name kubectl) /usr/local/bin/kubectl

# SSH into the control plane and run the following command
root@ubuntu-rke2-1:~# kubectl get nodes -o wide
NAME            STATUS   ROLES                       AGE   VERSION           INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
ubuntu-rke2-1   Ready    control-plane,etcd,master   73m   v1.26.14+rke2r1   143.244.139.181   <none>        Ubuntu 22.04.4 LTS   5.15.0-67-generic   containerd://1.7.11-k3s2
ubuntu-rke2-2   Ready    <none>                      20m   v1.26.14+rke2r1   143.244.136.174   <none>        Ubuntu 22.04.4 LTS   5.15.0-67-generic   containerd://1.7.11-k3s2
 

# Repeat the same with node 3 
kubectl get nodes -o wide

```

## Rancher

### Pre-requisite

* Refer to support matrix: https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-6-3/
* Install helm

```shell
curl -#L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify helm is installed and is all good to go
helm version

# add needed helm charts
# Rancher needs jetstack/cert-manager to create the self signed TLS certificates. We need to install it with the Custom Resource Definition (CRD). 
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add jetstack https://charts.jetstack.io

# update the distribution list
helm repo update

# helm install jetstack in the cert manager namespace
helm upgrade -i cert-manager jetstack/cert-manager -n cert-manager --create-namespace --set installCRDs=true
```

### Install rancher

```shell
# Installing rancher UI
export RANCHER_HOST_NAME=rke-1.techin48.com
helm upgrade -i rancher rancher-latest/rancher --create-namespace --namespace cattle-system --set hostname=$RANCHER_HOST_NAME --set bootstrapPassword=abcd123456 --set replicas=1

# After install, it should printout the following
Release "rancher" does not exist. Installing it now.
NAME: rancher
LAST DEPLOYED: Mon Mar 11 13:06:21 2024
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.
Check out our docs at https://rancher.com/docs/
If you provided your own bootstrap password during installation, browse to https://rke-1.techin48.com to get started.

If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:
echo https://rke-1.techin48.com/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')

To get just the bootstrap password on its own, run:
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'

# Verifying the installation is successful.
kubectl get pods -n cattle-system
NAME                       READY   STATUS    RESTARTS   AGE
helm-operation-lcj5q       2/2     Running   0          20s
```

### Accessing Rancher via the UI (Use your hostname)

* Click https://rke-1.techin48.com
* Login with the password set up
* Verify the node information
* Rancher works in a spoke and a hub model https://www.transvirtual.com/blog/what-is-the-hub-and-spoke-model/
* More information on different models of set up is available here https://ranchermanager.docs.rancher.com/

## Longhorn

### Install Longhorn

```shell
# get charts
helm repo add longhorn https://charts.longhorn.io

# update the distribution list
helm repo update

# install
helm upgrade -i longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
```
### Go back to the host being set up
* Click https://rke-1.techin48.com
* Longhorn should show up here and allow you to manage

## References

* https://github.com/clemenko/rke_install_blog
* https://www.youtube.com/watch?v=oM-6sd4KSmA 