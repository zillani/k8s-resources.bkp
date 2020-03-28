# Issues
 1. [Upgrade issue](#Upgrade-issue)
 2. [x509 certificate error](#x509-certificate-error)
 3. [resolv.cnf gets overridden](#resolv.cnf-gets-overridden)
 4. [ipaddress of master got updated](#ipaddress-of-master-got-updated)
 
## Upgrade issue

FATAL: this version of kubeadm only supports deploying clusters 
with the control plane version >= 1.16.0. Current version: v1.14.1

```bash
 You need to upgrade from 1.14 to 1.15 then 1.16 to 1.17
```

## x509 certificate error
Error:
x509: certificate signed by unknown authority 
(possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")

```bash
rm -rf ~/.kube/config
cat /etc/kubernetes/admin.conf > ~/.kube/config
```

## resolv.cnf gets overridden

![what overrides etc/resolv.conf](https://unix.stackexchange.com/questions/174349/what-overwrites-etc-resolv-conf-on-every-boot)
dont' forget to turnoff swap

## ipaddress of master got updated

![check this](https://github.com/kubernetes/kubeadm/issues/338#issuecomment-605435843)