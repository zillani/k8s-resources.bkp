# Security
## Table of Contents
1. [SecurityContext](#SecurityContext)
2. [Consuming Secrets](#Consuming-Secrets)
3. [ServiceAccounts](#ServiceAccounts)
4. [NetworkPolicy](#NetworkPolicy)
   1. [Network Policy with ingress and egress](#Network-Policy-with-ingress-and-egress)
   2. [Network Policy with ipBlock](#Network-Policy-with-ipBlock)

## SecurityContext

```bash
kubectl create -f second.yaml
kubectl exec -it secondApp -- sh
ps aux
grep Cap /proc/1/status
capsh --decode=00000000a80425fb
```
read more about kernel capabilities here, https://github.com/torvalds/linux/blob/master/include/uapi/linux/capability.h

Now, delete the pod & edit the second.yaml & add new capabilities, 

```bash
add: [ "NET_ADMIN" , "SYS_TIME" ]
```
Setting NET ADMIN to allow interface, routing, and other network conﬁguration. 
Setting SYS TIME, allows system clock conﬁguration.

Now, create the pod again & check Cap settings

```bash
kubectl create -f second.yaml
kubectl exec -it secondapp -- sh
grep Cap /proc/1/status
capsh --decode=00000000aa0435fb
```
You will observe that new capabilities are added.

## Consuming Secrets

```bash
echo "batman" |base64
YmF0bWFuCg==
kubectl create -f secret-second.yaml
kubectl exec -it secondapp -- /bin/sh
cat /mysqlpassword/password
cd /mysqlpassword
ls -lah
```
note that password file is a symbolic link to `data`

## ServiceAccounts


## NetworkPolicy

An early architecture decision with Kubernetes was non-isolation, that all pods were able to connect to all other pods and nodes by design. In more recent releases the use ofa NetworkPolicy allows for pod isolation. The policy only has effect when the networkplugin, like `ProjectCalico`, are capable of honoring them. If used with a plugin like `ﬂannel` they will have no effect. The use of matchLabels allows for more granular selection within the namespace which can be selected using a namespaceSelector. Using multiple labels can allow for complex application of rules. 

[network-policies](#https://kubernetes.io/docs/concepts/services-networking/network-policies)

### Network Policy with ingress and egress

Below app creates a policy that denies all traffic except from ingress/egress
```bash
kubectl create -f allclosed.yaml
```

Update `second.yaml` with the below config
```bash
containers: 
- name: webserver
  image: nginx 
- name: busy 
  image: busybox 
```
Observe the pod is failing `kubectl get event` , now remove the `securityContext` , add a label and deploy again, 
create the service to expose,

```bash
metadata: 
  name: secondapp 
  labels: 
    example: second
```

```bash
kubectl create service nodeport secondapp --tcp=80
```

Using this, edit to add the `selector` and a nodeport 

```bash
kubectl get svc secondapp -o yaml

```

```bash
...
ports:
- name: "80"
  nodePort: 32000
...
selector:
  example: second
```

Now test the service with nodeIp & clusterIP

Test ingress using node-ip & clusterIP with port,

```bash
curl <http://node-ip>
curl <clusterIP>:<nodePort>
```

Now test `egress` from container to oustide using `netcat`,

```bash
kubectl exec -it -c busy secondapp sh

nc -vz 127.0.0.1 80
127.0.0.1 (127.0.0.1:80) open
nc -vz www.linux.com 80
www.linux.com (151.101.185.5:80) open
exit
```

### Network Policy with ipBlock

Let's create a network policy that will allow to access the service
from private ipv4 16bit address like `192.168..`

- Test from your laptop `curl <nodeIp-public>:<nodePort>`
- Test from node `curl <nodeIp-private>`
- Test from container to external web by shelling into it & `netcat`

All of these will timeout,

__Let's Remove Egress and Add eth0__

Add below config to `allclosed.yaml`

```bash
 spec:
 podSelector: {} 
 policyTypes:
 - Ingress
 # - Egress
 
```

```bash
kubectl exec -it -c busy secondapp sh
nc -vz www.linux.com 80
www.linux.com (151.101.185.5:80) open
ip a
```
use the `eth0` range and update it as below,

```bash
policyTypes:
- Ingress 
ingress: #<-- Add this and following three lines 
- from:
  - ipBlock: 
      cidr: 192.168.0.0/16
# - Egress
```

Check the network policy, 
```bash
kubectl get network-policy
```

```bash
kubectl replace -f ~/allclosed.yaml

- Ingress 
ingress:
- from: 
  - ipBlock: 
      cidr: 192.168.0.0/16 
  ports: #<-- Add this and two following lines
  - port: 80 
    protocol: TCP 
# - Egres
```

```bash
kubectl replace -f allclosed.yaml
```

By adding the ip, you can access the service like, 

```bash
curl http://192.168.55.91
```

