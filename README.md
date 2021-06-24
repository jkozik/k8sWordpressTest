# Test Kubernetes setup by deploying a sample wordpress/mysql service
Clone this repository onto a kubectl client and apply the yaml files. This will deploy a wordpress pod and a mysql pod and exposes them as a NodePort service.  Then from a browser one can install a wordpress instance.
This is a really good test of the cluster and a good exercise of the basic kubectl commands.

I have host setup with a kubectl envornment with access rights to my cluster.  From this user login, run the following:
```
git clone https://github.com/jkozik/k8sWordpressTest
cd k8sWordpressTest
kubectl apply -f .
```
