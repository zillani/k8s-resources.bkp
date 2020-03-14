# Exposing applications
## Table of Contents
1. [Service](#Service)
2. [Ingress](#Ingress)
3. [Service Mesh](#Service-Mesh)

## Service

Service can be exposed in three ways,
- ClusterIP (only reacheable within the cluster)
- NodePort (access outside cluster) 
- LoadBalancer (assigns loadbalancer, works on cloudproviders)

Note: ClusterIp is the default service type.

```bash
kubectl create -f service.yaml
```

__Service Diagram__

![](https://raw.githubusercontent.com/zillani/img/master/k8s-resources/services-diagram.jpg)


The __kube-proxy__ running on cluster nodes watches the API server service resources. It presents a type of virtual IP address for services other than __ExternalName__. The mode for this process has changed over versions of Kubernetes. 

In `v1.0`, services ran in userspace mode as `TCP/UDP` over `IP` or Layer 4. 
In the `v1.1` release, the iptables proxy was added and became the default mode starting with `v1.2`. 
In the iptables proxy mode, kube-proxy continues to monitor the API server for changes in Service and Endpoint objects, and updates rules for each object when created or removed. One limitation to the new mode is an inability to connect to a Pod should the original request fail, so it uses a Readiness Probe to ensure all containers are functional prior to connection. This mode allows for up to approximately 5000 nodes. Assuming multiple Services and Pods per node, this leads to a bottleneck in the kernel.
Another mode beginning in `v1.9` is __ipvs__. While in beta, and expected to change, it works in the kernel space for greater speed, and allows for a configurable load-balancing algorithm, such as round-robin, shortest expected delay, least connection and several others. This can be helpful for large clusters, much past the previous 5000 node limitation. This mode assumes IPVS kernel modules are installed and running prior to kube-proxy. 
The kube-proxy mode is configured via a flag sent during initialization, such as __mode=iptables__ and could also be __IPVS__ or __userspace__. 

## Ingress

### Ingress resource

An ingress resource is an API object containing a list of rules matched against all incoming requests. Only HTTP rules are currently supported. In order for the controller to direct traffic to the backend, the HTTP request must match both the host and the path declared in the ingress

### Ingress Controller

![](https://raw.githubusercontent.com/zillani/img/master/k8s-resources/ingress-controller.jpg)

Handling a few services can be easily done. However, managing thousands or tens of thousands of services can create inefficiencies. The use of an Ingress Controller manages ingress rules to route traffic to existing services. Ingress can be used for fan out to services, name-based hosting, TLS, or load balancing. Another feature is the ability to expose low-numbered ports. Services have been hard-coded not to expose ports lower than 1024.
There are a few Ingress Controllers with nginx and GCE that are "officially supported" by the community. Traefik (pronounced "traffic") and HAProxy are in common use, as well. More controllers are planned, as is support for more HTTPS/TLS modes, combining L4 and L7 ingress and requesting name or IP via claims.


If you have large number of services to be exposed outside the cluster, you need ingress. 
we are using traefic ingress because it's easier than nginx & GCE. 

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
```

## Service Mesh
A service mesh consists of edge and embedded proxies communicating with each other and handing traffic based on rules from a control plane.
For more complex connections or resources such as __service discovery__, __rate limiting__, __traffic management__ and __advanced metrics__ you may want to implement a service mesh.

### Istio
A powerful tool set which leverages Envoy proxies via a multi-component control plane.

### Linkerd
An easy to deploy, fast, and ultralight service mesh.
Linkerd has default __linkerd2-proxy__ (aka linkerd-proxy) in it's dataplane.
Can be configured to use __Envoy__ proxy also.

### Envoy
Envoy is a proxy, not service mesh. Both linkerd & istio can be configured to use envoy proxy. 


### Istio service mesh

![](https://raw.githubusercontent.com/zillani/img/master/k8s-resources/istio-arch.jpg)

### Linkerd service mesh

![](https://raw.githubusercontent.com/zillani/img/master/k8s-resources/linkerd2-arch.jpg)

#### Control Plane vs Data Plane
- __The Control Plane__ responsibility is to manage and configure the sidecar proxies to enforce policies and collect telemetry.
- __The Data Plane__ responsibility is to handle the communication between the services and take care of the functionalities like Service Discovery, Load Balancing, Traffic Management, Health Check, etc

