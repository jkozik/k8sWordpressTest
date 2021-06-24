# Test Kubernetes setup by deploying a sample wordpress/mysql service
Clone this repository onto a kubectl client and apply the yaml files. This will deploy a wordpress pod and a mysql pod and exposes them as a NodePort service.  Then from a browser one can install a wordpress instance.  Further, this repository includes a Persistent Volume that is created to point to an NFS share outside of the cluster. 

This is a really good test of the cluster and a good exercise of the basic kubectl commands.

First, on a host external to the cluster verify that the /var/nfsshare is setup to include two directories: html and mysql.  These directories are used by wordpress and mysql. Further verify that the cluster worker nodes are configured with NFS client software -- [NFS server / client setup](https://github.com/jkozik/SetupKubeadmCentos7/blob/master/ClusterBasics.md#volume-storage-nfs-pv-pvc)

From 

I have a host setup with a kubectl environment with access rights to my cluster.  From this user login, check the nfsshare:
```
[jkozik@dell2 wptest]$ ls /var/nfsshare
html mysql
```
Next, pull in the yaml files for this repository.  
```
git clone https://github.com/jkozik/k8sWordpressTest
cd k8sWordpressTest
ls
[jkozik@dell2 k8sWordpressTest]$ ls
mysql-deploy.yml  pvc-mysql.yml  pvc-wordpress.yml  pv-wordpress-mysql.yml  README  README.md  secret.yml  wordpress-deploy.yml
[jkozik@dell2 k8sWordpressTest]$
```
## Persistent Volume
For starters, get the presistent volume setup.  Verify that the specifics in the file pv-wordpress-mysql.yml nfs sections are appropriately customized:
```
vi pv-wordpress-mysql.yml
kind: PersistentVolume
metadata:
  name: wordpress-persistent-storage
  labels:
    app: wordpress
    tier: frontend
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.101.152
    # Exported path of your NFS server
    path: "/var/nfsshare/html"

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-persistent-storage
  labels:
    app: wordpress
    tier: mysql
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.101.152
    # Exported path of your NFS server
    path: "/var/nfsshare/mysql"
```
Next, bringup the persistent volumes by applying the following command and verifying that the PV<->PVC binds correctly.
```
kubectl apply -f pv-wordpress-mysql.yml
kubectl apply -f pvc-wordpress.yml
kubectl apply -f pvc-mysql.yml

[jkozik@dell2 ~]$ kubectl get pv,pvc
NAME                                            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                  STORAGECLASS   REASON   AGE
persistentvolume/mysql-persistent-storage       1Gi        RWX            Retain           Bound    default/mysql-persistent-storage                               42h
persistentvolume/wordpress-persistent-storage   1Gi        RWX            Retain           Bound    default/wordpress-persistent-storage                           42h

NAME                                                 STATUS   VOLUME                         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql-persistent-storage       Bound    mysql-persistent-storage       1Gi        RWX                           42h
persistentvolumeclaim/wordpress-persistent-storage   Bound    wordpress-persistent-storage   1Gi        RWX                           42h
[jkozik@dell2 ~]$
```
## Deploy Mysql with Secret
Next, wordpress needs a password to talk to mysql. The secret.yml file needs to be applyed to create the password secret resource.  Then deploy the mysql pod and expose it as a service.
```
kubectl apply -f secret.yml
kubectl apply -f mysql-deploy.yml
# verify the deployment worked
kubectl get deployment,service
kubectl get pods -o wide
kubectl logs wordpress-mysql-xxxx-yyyy
kubectl describe pods wordpress-mysql-xxxx-yyyy
```
I put a break here, because when I first did this, I had troubles with my nfs mounts that did not show up with PV, PVC steps.  The kubectl logs command gave me good diagnostics.

## Deploy Wordpress
Next, apply the wordpress deployment yaml file and expose it as a NodePort service. Key point: you need a NodePort service inorder to be able externally access the service.  The mysql service is a ClusterIP -- only visible at the cluster level.
```
kubectl apply -f wordpress-deploy.yml
kubectl get deployment,service
kubectl get pods --all-namespaces -o wide
kubectl describe pods wordpress-xxxx-yyyy
kubectl logs pods wordpress-xxxx-yyyy
```
## Test in browser
From here, go to a browser and access http://192.168.100.172:3xxxx. This should be the wordpress installation screen.

# References
This was done following the excellent HowTo below.  Note, there were a couple steps that didn't work.  I fixed the minor typos in my writeup above.
- https://medium.com/@containerum/how-to-deploy-wordpress-and-mysql-on-kubernetes-bda9a3fdd2d5
- 
