# Architecture

1. [Architecture](#Architecture)
2. [Installation](#Installation)

## Architecture

![](https://raw.githubusercontent.com/zillani/img/master/k8s-resources/k8s-arch.png)

## Installation
- install vmware 
- install 3 ubuntu servers 18.x (say, ubuntu1, ubuntu2, ubuntu3)
- ubuntu1 is master node and its should have 2 vCPUs, other can have 1vCPU
- install k8s components on master node using this script, k8sMaster.sh
- once the installation is done, in the end you can find the output command generated
- for worker nodes, use k8sSecond.sh
- copy the command generated on master node and exectue the command on worker node
- after the installation is complete, you can ssh into the master node and access the cluster, 
  OR you can copy the config located at `/etc/kubernetes/admin.conf`

##### note: If you are accessing the cluster after restarting the nodes, please turn off swap
`sudo swapoff -a`




