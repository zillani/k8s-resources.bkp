#### install vmware

#### install ubuntu on vmware

#### install k8s
install k8s components on master node using this script, k8sMaster.sh
for worker nodes, use k8sSecond.sh

##### note: Please make sure to turnoff swap if you restart the nodes
`sudo swapoff -a`

#### config
after the installation is complete, you can ssh into the master node and access the cluster, 
OR you can copy the config located at `/etc/kubernetes/admin.conf`


