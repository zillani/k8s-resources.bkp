# Deployment Configuration
## Table of Contents
1. [Create configmap](#Create-configmap)


## Create configmap

```
mkdir colors
echo r > colors/red
echo b > colors/blue
echo y > colors/yellow
echo g > colors/green
```
Now, create the config map using the file, 
```
kubectl create cm colors --from-file=./colors
kubectl get cm colors -o yaml
```

