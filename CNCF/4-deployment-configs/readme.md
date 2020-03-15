# Deployment Configuration
## Table of Contents
1. [Storage](#Volumes)
   1. [Volumes](#Volumes)
   2. [Volume Types](#Volume-Types)
   3. [Shared Volume Example](#Shared-Volume-Example)
   4. [Persistence Volumes and Claims](#Persistence-Volumes-and-Claims)
      1. [Persistence Volumes](#Persistence-Volumes)
      2. [Persistence Volume Claim](#Persistence-Volume-Claim)
   5. [Dynamic Provisioning](#Dynamic-Provisioning)
2. [Secrets](#Secrets)
   1. [Secrets via env variables](#Secrets-via-env-variables)
   2. [Secrets as volumes](#Secrets-as-volumes)
3. [Configmap](#Configmap)
   1. [Create configmap](#Create-configmap)
   2. [Portable configmaps](#Portable-configmaps)
   3. [Using configmaps](#Using-configmaps)
4. [Deployment Configuration Status](#Deployment-Configuration-Status)
5. [Scaling and Rolling Updates](#Scaling-and-Rolling-Updates)
6. [Deployment Rollbacks](#Deployment-Rollbacks)



## Volumes

Containers are considered __transient__ and this could lead to a loss of data when it terminates. 

A __volume__ can persist longer than a pod, and can be accessed by multiple pods, using `PersistentVolumeClaims`. This allows for state persistency.
A volume is a directory, possibly pre-populated, made available to containers in a Pod. The creation of the directory, the backend storage of the data and the contents depend on the volume type. 

As of `v1.14`, there are 28 different volume types ranging from rbd for Ceph, to NFS, to dynamic volumes from a cloud provider like Google's gcePersistentDisk. Each has particular configuration options and dependencies. 

An alpha feature to `v1.9` is the __Container Storage Interface (CSI)__ with the goal of an industry standard interface for container orchestration to allow access to arbitrary storage systems.

Currently, volume plugins are __in-tree__, meaning they are compiled and built with the core Kubernetes binaries. This __out-of-tree__ object will allow storage vendors to develop a single driver and allow the plugin to be containerized. This will replace the existing __Flex__ plugin which requires elevated access to the host node, a large security concern. 
Should you want your storage lifetime to be distinct from a Pod, you can use Persistent Volumes. These allow for empty or pre-populated volumes to be claimed by a Pod using a Persistent Volume Claim, then outlive the Pod. Data inside the volume could then be used by another Pod, or as a means of retrieving data.


### Volumes 

![](https://raw.githubusercontent.com/zillani/img/master/k8s-resources/volumes.jpg)

A Pod specification can declare one or more volumes and where they are made available.Keeping acquired data or ingesting it into other containers is a common task, typically requiring the use of a Persistent Volume Claim (PVC).
The same volume can be made available to multiple containers within a Pod, which can be a method of `container-to-container communication.` 
A volume can be made available to multiple Pods, with each given an access mode to write. There is no concurrency checking, which means data corruption is probable, unless outside locking takes place


A particular access mode is part of a Pod request. As a request, the user may be granted more, but not less access, though a direct match is attempted first. The cluster groups volumes with the same mode together, then sorts volumes by size, from smallest to largest. The claim is checked against each in that access mode group, until a volume of sufficient size matches. The three access modes are RWO (ReadWriteOnce), which allows read-write by a single node, ROX (ReadOnlyMany), which allows read-only by multiple nodes, and RWX (ReadWriteMany), which allows read-write by many nodes. 
When a volume is requested, the local kubelet uses the kubelet_pods.go script to map the raw devices, determine and make the mount point for the container, then create the symbolic link on the host node filesystem to associate the storage to the container. The API server makes a request for the storage to the StorageClass plugin, but the specifics of the requests to the backend storage depend on the plugin in use. 
If a request for a particular StorageClass was not made, then the only parameters used will be access mode and size. The volume could come from any of the storage types available, and there is no configuration to determine which of the available ones will be used. 

### Volume Types

There are several types that you can use to define volumes, each with their pros and cons. Some are local, and many make use of network-based resources.
In GCE or AWS, you can use volumes of type GCEpersistentDisk or awsElasticBlockStore, which allows you to mount GCE and EBS disks in your Pods, assuming you have already set up accounts and privileges.
emptyDir and hostPath volumes are easy to use. As mentioned, emptyDir is an empty directory that gets erased when the Pod dies, but is recreated when the container restarts. The hostPath volume mounts a resource from the host node filesystem. The resource could be a directory, file socket, character, or block device. These resources must already exist on the host to be used. There are two types, DirectoryOrCreate and FileOrCreate, which create the resources on the host, and use them if they don't already exist. 
NFS (Network File System) and iSCSI (Internet Small Computer System Interface) are straightforward choices for multiple readers scenarios.
rbd for block storage or CephFS and GlusterFS, if available in your Kubernetes cluster, can be a good choice for multiple writer needs.
Besides the volume types we just mentioned, there are many other possible, with more being added: azureDisk, azureFile, csi, downwardAPI, fc (fibre channel), flocker, gitRepo, local, projected, portworxVolume, quobyte, scaleIO, secret, storageos, vsphereVolume, persistentVolumeClaim, etc.​

### Shared Volume Example

The following YAML file creates a pod with two containers, both with access to a shared volume:
```bash
containers:
       - image: busybox
     volumeMounts:
       - mountPath: /busy
     name: test
     name: busy
       - image: busybox
     volumeMounts:
       - mountPath: /box
     name: test
     name: box
     volumes:
       - name: test
       emptyDir: {} 
```

Now, let's try to create a file in the mountPath from one container & access it from another container,

```bash
kubectl exec -ti busybox -c box -- touch /box/foobar 
kubectl exec -ti busybox -c busy -- ls /busy total 0
foobar
```

### Persistence Volumes and Claims

A __persistent volume (pv)__ is a storage abstraction used to retain data longer than the Pod using it. 
Pods define a volume of type __persistentVolumeClaim (pvc)__ with various parameters for size and possibly the type of backend storage known as its StorageClass. The cluster then attaches the persistentVolume. 
Kubernetes will dynamically use volumes that are available, irrespective of its storage type, allowing claims to any backend storage. 
There are several phases to persistent storage: 
Provisioning can be from pvs created in advance by the cluster administrator, or requested from a dynamic source, such as the cloud provider. 

Binding occurs when a control loop on the master notices the PVC, containing an amount of storage, access request, and optionally, a particular StorageClass. The watcher locates a matching PV or waits for the StorageClass provisioner to create one. The pv must match at least the storage amount requested, but may provide more. 
The use phase begins when the bound volume is mounted for the Pod to use, which continues as long as the Pod requires. 
Releasing happens when the Pod is done with the volume and an API request is sent, deleting the PVC. The volume remains in the state from when the claim is deleted until available to a new claim. The resident data remains depending on the persistentVolumeReclaimPolicy. 

The reclaim phase has three options:
- Retain, which keeps the data intact, allowing for an administrator to handle the storage and data.
- Delete tells the volume plugin to delete the API object, as well as the storage behind it.
- The Recycle option runs an rm -rf /mountpoint and then makes it available to a new claim. With the stability of dynamic provisioning, the Recycle option is planned to be deprecated.

```bash
kubectl get pv
kubectl get pvc
```

### Persistence Volume

```bash
kind: PersistentVolume 
apiVersion: v1 
metadata: 
    name: 10Gpv01 
    labels:
         type: local 
spec: 
    capacity: 
        storage: 10Gi 
    accessModes: 
        - ReadWriteOnce 
    hostPath: 
        path: "/somepath/data01"
```
Each type will have its own configuration settings. For example, an already created Ceph or GCE Persistent Disk would not need to be configured, but could be claimed from the provider. 
Persistent volumes are cluster-scoped, but persistent volume claims are namespace-scoped. An alpha feature since v1.11, this allows for static provisioning of Raw Block Volumes, which currently support the Fibre Channel plugin. There is a lot of development and change in this area, with plugins adding dynamic provisioning

### Persistence Volume Claim

With a persistent volume created in your cluster, you can then write a manifest for a claim and use that claim in your pod definition

```bash
kind: PersistentVolumeClaim
apiVersion: v1 
metadata: 
    name: myclaim
spec: 
    accessModes: 
        - ReadWriteOnce
    resources:
      requests:
        storage: 8GI
```

Config in the Pod,

```bash
spec:
  containers:
....
  volumes:
    - name: test-volume
    persistentVolumeClaim:
      claimName: myclaim
```

Another complex config,

Point to ponder: If you had one application ingesting data, but also want to archive, ingest into a data lake, and forward the data, how would you use persistent volume claims?

```bash
volumeMounts:
      - name: Cephpd
        mountPath: /data/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
        - '10.19.14.22:6789'
        - '10.19.14.23:6789'
        - '10.19.14.24:6789'
        pool: k8s
        image: client
        fsType: ext4
        readOnly: true
        user: admin
        keyring: /etc/ceph/keyring
        imageformat: "2"
        imagefeatures: "layering"
```

### Dynamic Provisioning

While handling volumes with a persistent volume definition and abstracting the storage provider using a claim is powerful, a cluster administrator still needs to create those volumes in the first place. Starting with Kubernetes `v1.4`, __Dynamic Provisioning__ allowed for the cluster to request storage from an exterior, pre-configured source. API calls made by the appropriate plugin allow for a wide range of dynamic storage use. 

The __StorageClass__ API resource allows an administrator to define a persistent volume provisioner of a certain type, passing storage-specific parameters.
With a StorageClass created, a user can request a claim, which the API Server fills via auto-provisioning. The resource will also be reclaimed as configured by the provider. AWS and GCE are common choices for dynamic storage, but other options exist, such as a Ceph cluster or iSCSI. Single, default class is possible via annotation.

#### Storage class using GCE,

```bash
apiVersion: storage.k8s.io/v1        # Recently became stable
kind: StorageClass
metadata:
  name: fast                         
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd 
```

## Secrets

Pods can access local data using volumes, but there is some data you don't want readable to the naked eye. Passwords may be an example. Someone reading through a YAML file may read a password and remember it. Using the Secret API resource, the same password could be encoded. A casual reading would not give away the password. 
You can create, get, or delete secrets:
```bash 
kubectl get secrets
```
Secrets can be manually encoded with kubectl create secret:​

```bash
kubectl create secret generic --help 
kubectl create secret generic mysql --from-literal=password=root
```
A secret is not encrypted by default, only base64-encoded. You can see the encoded string inside the secret with kubectl. The secret will be decoded and be presented as a string saved to a file. The file can be used as an environmental variable or in a new directory, similar to the presentation of a volume.

In order to encrypt secrets, you must create an `EncryptionConfiguration` object with a key and proper identity. Then, the kube-apiserver needs the `--encryption-provider-config` flag set to a previously configured provider, such as aescbc or ksm. Once this is enabled, you need to recreate every secret, as they are encrypted upon write. Multiple keys are possible. Each key for a provider is tried during decryption. The first key of the first provider is used for encryption. To rotate keys, first create a new key, restart (all) kube-apiserver processes, then recreate every secret.

```bash
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  password: TEZUckAxbgo= 
```
### Secrets via env variables

There is no limit to the number of Secrets used, but there is a __1MB__ limit to their size. Each secret occupies memory, along with other API objects, so very large numbers of secrets could deplete memory on a host.
They are stored in the tmpfs storage on the host node, and are only sent to the host running Pod. All volumes requested by a Pod must be mounted before the containers within the Pod are started. So, a secret must exist prior to being requested. 

```bash
spec: 
  containers: 
  - image: mysql:5.5 
    env: 
    - name: MYSQL_ROOT_PASSWORD 
      valueFrom: 
        secretKeyRef: 
          name: mysql 
          key: password 
    name: mysql
```

### Secrets as volumes

Create a secret and mount it to path, 

```bash
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
data:
  password: TEZUckAxbgo= 
```

use this secret `mysql-pass` below,

```bash
... 
spec: 
    containers: 
    - image: busybox 
      command: 
        - sleep 
        - "3600" 
      volumeMounts: 
      - mountPath: /mysqlpassword 
        name: mysql 
      name: busy 
    volumes: 
    - name: mysql 
        secret: 
          secretName: mysql-pass
```

## Configmap

There are two API Objects which exist to provide data to a Pod already. Encoded data can be passed using a `Secret` and non-encoded data can be passed with a `ConfigMap`. These can be used to pass important data like SSH keys, passwords, or even a configuration file like `/etc/hosts`

### Create configmap
```bash
mkdir colors
echo r > colors/red
echo b > colors/blue
echo y > colors/yellow
echo g > colors/green
```
Now, create the config map using the file, 

```bash
kubectl create cm colors --from-file=./colors
kubectl get cm colors -o yaml
```

### Portable Configmaps

A similar API resource to Secrets is the ConfigMap, except the data is not encoded. In keeping with the concept of decoupling in Kubernetes, using a ConfigMap decouples a container image from configuration artifacts. 
They store data as sets of key-value pairs or plain configuration files in any format. The data can come from a collection of files or all files in a directory. It can also be populated from a literal value. 

A ConfigMap can be used in several different ways. A Pod can use the data as environmental variables from one or more sources. The values contained inside can be passed to commands inside the pod. A Volume or a file in a Volume can be created, including different names and particular access modes. In addition, cluster components like controllers can use the data. ​

Let's say you have a file on your local filesystem called __config.js__. You can create a ConfigMap that contains this file. The configmap object will have a __data__ section containing the content of the file:

```bash
kubectl get configmap foobar -o yaml 
```

```bash
kind: ConfigMap 
apiVersion: v1 
metadata: 
    name: foobar 
data: 
    config.js: | 
        { 
```

ConfigMaps can be consumed in various ways:
- Pod environmental variables from single or multiple ConfigMaps
- Use ConfigMap values in Pod commands
- Populate Volume from ConfigMap
- Add ConfigMap data to specific path in Volume
- Set file names and access mode in Volume from ConfigMap data
- Can be used by system components and controllers.

### Using configmaps

Like secrets, you can use ConfigMaps as environment variables or using a volume mount. They must exist prior to being used by a Pod, unless marked as __optional__. They also reside in a specific namespace.
In the case of environment variables, your pod manifest will use the __valueFrom__ key and the __configMapKeyRef__ value to read the values. For instance:

```bash
env: 
- name: SPECIAL_LEVEL_KEY 
  valueFrom: 
    configMapKeyRef: 
      name: special-config 
      key: special.how
```

With volumes, you define a volume with the __configMap__ type in your pod and mount it where it needs to be used.

```bash
volumes: 
    - name: config-volume 
      configMap: 
        name: special-config
```

## Deployment Configuration Status

The Status output is generated when the information is requested: ​
```bash
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2017-12-21T13:57:07Z
    lastUpdateTime: 2017-12-21T13:57:07Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 2
  readyReplicas: 2
  replicas: 2
  updatedReplicas: 2 
```

The output above shows what the same deployment were to look like if the number of replicas were increased to two. The times are different than when the deployment was first generated. 

#### availableReplicas :

Indicates how many were configured by the ReplicaSet. This would be compared to the later value of __readyReplicas__, which would be used to determine if all replicas have been fully generated and without error. 

#### observedGeneration :

Shows how often the deployment has been updated. This information can be used to understand the rollout and rollback situation of the deployment. 

## Scaling and Rolling Updates

The API server allows for the configurations settings to be updated for most values. There are some immutable values, which may be different depending on the version of Kubernetes you have deployed. 
A common update is to change the number of replicas running. If this number is set to zero, there would be no containers, but there would still be a ReplicaSet and Deployment. This is the backend process when a Deployment is deleted. 

```bash
kubectl scale deploy/dev-web --replicas=4
deployment "dev-web" scaled 
```
```bash
kubectl get deployments
NAME     DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
dev-web  4        4        4           1          12m 
```
Non-immutable values can be edited via a text editor, as well. Use edit to trigger an update. For example, to change the deployed version of the nginx web server to an older version: 

```bash
kubectl edit deployment nginx
....
      containers:
      - image: nginx:1.8 #<<---Set to an older version
        imagePullPolicy: IfNotPresent
        name: dev-web 
.... 
```

This would trigger a rolling update of the deployment. While the deployment would show an older age, a review of the Pods would show a recent update and older version of the web server application deployed.

## Deployment Rollbacks

With all the ReplicaSets of a Deployment being kept, you can also roll back to a previous revision by scaling up and down the ReplicaSets the other way. Next, we will have a closer look at rollbacks, using the --record option of the kubectl command, which allows annotation in the resource definition. The create generator does not have a record function.

```bash
kubectl set image deployment ghost --image=ghost:0.9 --record
kubectl get deployments ghost -o yaml
```
```bash
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubernetes.io/change-cause: kubectl set image deployment ghost --image=ghost:0.9 --record
```
Should an update fail, due to an improper image version,

```bash
kubectl set image deployment/ghost ghost=ghost:0.9 --all
kubectl rollout history deployment/ghost deployments "ghost":

REVISION     CHANGE-CAUSE
1            <none>
2            kubectl set image deployment/ghost ghost=ghost:0.9 --all
```
you can roll back the change to a working version with
`kubectl rollout undo`

```bash
$ kubectl get pods
NAME                       READY    STATUS               RESTARTS    AGE 
ghost-2141819201-tcths     0/1      ImagePullBackOff     0           1m​
```

```bash
kubectl rollout undo deployment/ghost
```

You can roll back to a specific revision with the `--to-revision=2` option. 
You can also edit a Deployment using the kubectl edit command. 
You can also pause a Deployment, and then resume.

```bash
kubectl rollout pause deployment/ghost
​kubectl rollout resume deployment/ghost
```
Please note that you can still do a rolling update on ReplicationControllers with the __kubectl rolling-update__ command, but this is done on the client side. Hence, if you close your client, the rolling update will stop. 