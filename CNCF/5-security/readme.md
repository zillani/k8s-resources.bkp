# Security
## Table of Contents
1. [SecurityContext](#SecurityContext)
2. [Consuming Secrets](#Consuming-Secrets)
3. [ServiceAccounts](#ServiceAccounts)
4. [NetworkPolicy](#NetworkPolicy)
5. [Testing the policy](#Testing-the-policy)

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



