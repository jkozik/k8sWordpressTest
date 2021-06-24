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
[jkozik@dell2 k8sWordpressTest]$ kubectl apply -f pv-wordpress-mysql.yml
persistentvolume/wordpress-persistent-storage created
persistentvolume/mysql-persistent-storage created

[jkozik@dell2 k8sWordpressTest]$ kubectl apply -f pvc-wordpress.yml
persistentvolumeclaim/wordpress-persistent-storage created

[jkozik@dell2 k8sWordpressTest]$ kubectl apply -f pvc-mysql.yml
persistentvolumeclaim/mysql-persistent-storage created

[jkozik@dell2 k8sWordpressTest]$ kubectl get pv,pvc
NAME                                            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                  STORAGECLASS   REASON   AGE
persistentvolume/mysql-persistent-storage       1Gi        RWX            Retain           Bound    default/mysql-persistent-storage                               67s
persistentvolume/wordpress-persistent-storage   1Gi        RWX            Retain           Bound    default/wordpress-persistent-storage                           67s

NAME                                                 STATUS   VOLUME                         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql-persistent-storage       Bound    mysql-persistent-storage       1Gi        RWX                           45s
persistentvolumeclaim/wordpress-persistent-storage   Bound    wordpress-persistent-storage   1Gi        RWX                           55s
```
## Deploy Mysql with Secret
Next, wordpress needs a password to talk to mysql. The secret.yml file needs to be applyed to create the password secret resource.  Then deploy the mysql pod and expose it as a service.
```
[jkozik@dell2 k8sWordpressTest]$ kubectl delete secret mysql-pass
secret "mysql-pass" deleted

[jkozik@dell2 k8sWordpressTest]$ kubectl apply -f secret.yml
secret/mysql-pass created

[jkozik@dell2 k8sWordpressTest]$ kubectl apply -f mysql-deploy.yml
service/wordpress-mysql created
deployment.apps/wordpress-mysql created

[jkozik@dell2 k8sWordpressTest]$ # verify the deployment worked
[jkozik@dell2 k8sWordpressTest]$ kubectl get deployment,service
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kubernetes-bootcamp   1/1     1            1           20d
deployment.apps/wordpress-mysql       1/1     1            1           16s

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes        ClusterIP   10.96.0.1      <none>        443/TCP        21d
service/wordpress-mysql   ClusterIP   None           <none>        3306/TCP       16s

[jkozik@dell2 k8sWordpressTest]$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
wordpress-mysql-66cb796566-kjhgd       1/1     Running   0          37s   10.68.77.133   kworker2   <none>           <none>
[jkozik@dell2 k8sWordpressTest]$ kubectl describe pods wordpress-mysql-66cb796566-kjhgd
Name:         wordpress-mysql-66cb796566-kjhgd
...
    Environment:
      MYSQL_ROOT_PASSWORD:  <set to the key 'password' in secret 'mysql-pass'>  Optional: false
    Mounts:
      /var/lib/mysql from mysql-persistent-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rpshm (ro)
...
Volumes:
  mysql-persistent-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  mysql-persistent-storage
    ReadOnly:   false
... 
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m20s  default-scheduler  Successfully assigned default/wordpress-mysql-66cb796566-kjhgd to kworker2
  Normal  Pulled     2m11s  kubelet, kworker2  Container image "mysql:5.6" already present on machine
  Normal  Created    2m11s  kubelet, kworker2  Created container mysql
  Normal  Started    2m10s  kubelet, kworker2  Started container mysql

```
I put a break here, because when I first did this, I had troubles with my nfs mounts that did not show up with PV, PVC steps.  The kubectl logs command gave me good diagnostics.

## Deploy Wordpress
Next, apply the wordpress deployment yaml file and expose it as a NodePort service. Key point: you need a NodePort service inorder to be able externally access the service.  The mysql service is a ClusterIP -- only visible at the cluster level.
```
[jkozik@dell2 k8sWordpressTest]$ kubectl apply -f wordpress-deploy.yml
service/wordpress created
deployment.apps/wordpress created

[jkozik@dell2 k8sWordpressTest]$ kubectl get deployment,service
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kubernetes-bootcamp   1/1     1            1           20d
deployment.apps/nginx-deployment      1/1     1            1           26h
deployment.apps/wordpress             1/1     1            1           8s
deployment.apps/wordpress-mysql       0/1     1            0           7m22s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP        21d
service/wordpress         NodePort    10.105.26.145   <none>        80:30622/TCP   8s
service/wordpress-mysql   ClusterIP   None            <none>        3306/TCP       7m22s

[jkozik@dell2 k8sWordpressTest]$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE    IP             NODE       NOMINATED NODE   READINESS GATES
wordpress-6db6fb5444-5kd7p             1/1     Running   1          49s    10.68.41.141   kworker1   <none>           <none>
wordpress-mysql-66cb796566-kjhgd       1/1     Running   0          8m3s   10.68.77.133   kworker2   <none>           <none>

```
## Test in browser
From here, go to a browser and access http://192.168.100.172:30622. This should be the wordpress installation screen.

# References
This was done following the excellent HowTo below.  Note, there were a couple steps that didn't work.  I fixed the minor typos in my writeup above.
- https://medium.com/@containerum/how-to-deploy-wordpress-and-mysql-on-kubernetes-bda9a3fdd2d5
- 
