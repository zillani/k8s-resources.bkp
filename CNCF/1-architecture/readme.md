# Architecture

1. [Architecture](#Architecture)
2. [Networking](#Networking)
   1. [Ingress](#Ingress)
   2. [Calico](#Calico)
   3. [CNI](#CNI)
2. [Installation](#Installation)

## Architecture

![](https://raw.githubusercontent.com/zillani/img/master/k8s-resources/k8s-arch.png)

- __kube-apiserver__: An agent
- __etcd__: stores cluster state (b-tree key-value)

- __kube-scheduler__: Sees request coming to API & finds suitable node to run container
- __kubelet__: The HEAVY LIFTER, accepts request for PodSpec (Json/Yaml) & manages resources
               & watches over them
- __kubeproxy__: Manages n/w rules & expose container on network

_note:_ from 1.15.1 multi-master is possible using stacked etcd or external database cluster

- __kube-controller-manager__: If state of cluster doesn't match the desired state, then it will 
							   controllers to match the desired state

## Networking

Pod is a co-located containers which share same IPAddress

Three communication challenges,
| Problem        | Solved by  |
| -------------  |:---------: |
| Coupled container-to-container | Pod Concept |
| Pod-to-Pod | Networking Config |
| External-to-Pod | Ingress |


### Ingress
Ingress controller or ServiceMesh (ISTIO, Linkerd) can be used to connect external traffic to pod.

Logger Pod: Sidecar for logging & writing to database
Pause Pod: To Retrieve namespace & IpAddress

![](https://raw.githubusercontent.com/zillani/img/master/k8s-resources/k8s-nw.jpg)

### Calico
Observe Calico pods running on BIRD protocol
![](https://raw.githubusercontent.com/zillani/img/master/k8s-resources/k8s-calico.jpg)

### CNI

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

_note_: If you are accessing the cluster after restarting the nodes, please turn off swap
`sudo swapoff -a`




