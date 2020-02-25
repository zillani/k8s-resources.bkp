# Exposing applications
## Table of Contents
1. [Service](#Service)
2. [Ingress](#Ingress)

## Service

Service can be exposed in three ways,
- ClusterIP (only reacheable within the cluster)
- NodePort (access outside cluster) 
- LoadBalancer (assigns loadbalancer, works on cloudproviders)

Note: ClusterIp is the default service type.

```bash
kubectl create -f service.yaml

```

## Ingress

If you have large number of services to be exposed outside the cluster you need ingress. 
we are using traefic ingress because its easier than nginx & GCE. 

```bash
kubectl create -f ingress-rbac.yaml
wget https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-ds.yaml
```
After downloading the traefik, remove the `securityContext` config and add `hostNetwork: true`

```bash
kubectl create -f traefik-ds.yaml
```
Now, ingest a ingress rule,

```bash
kubectl create -f ingress.rule.yaml
ip a
curl -H "Host: www.example.com" http://10.128.0.11"
```
Also try using the nodes' public ip

```bash
curl -H "Host: www.example.com" http://35.193.3.179
```

Create the third page app, 

```bash
kubectl run thirdpage --generator=run-pod/v1 --image=nginx --port=80 -l example=third
kubectl expose pod thirdpage --type=NodePort
kubectl exec -it thirdpage -- /bin/bash
```
After shell into the container, edit the nginx title
```bash
apt-get update
apt-get install vim -y
vim /usr/share/nginx/html/index.html
```
Now edit the ingress rule, update the host to thirdpage.org
```bash
kubectl edit ingress ingress-test
curl -H "Host: thirdpage.org" http://10.128.0.7/


