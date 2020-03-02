# Build
## Table of Contents
1. [Simple app](#simple-app)
2. [Build local docker registry](#build-local-docker-registry)
	1. [docker-compose and registry](#docker-compose-and-registry)
	2. [kompose](#kompose)


## Simple app

```
apt-get install python
docker build -t simpleapp
find / -name date.out
```

## Build local docker registry

#### docker-compose and registry
```
apt-get install docker-compose apache2-utils
mkdir -p /localdocker/data
cd /localdocker/
docker-compose up
curl http://localhost:5000/v2/ #don't forget / in the end
```


#### kompose
```
curl -L https://github.com/kubernetes/kompose/releases/download/v1.19.0/kompose-linux-amd64 -o kompose
chmod +x kompose
mv kompose /usr/local/bin
```