# LAB: Setting up wordpress with kind

In this LAB we will setup [Wordpress](https://wordpress.org/) on a local kubernetes cluster. To easily run a local
Kubernetes cluster we will use [kind](https://kind.sigs.k8s.io/).

kind is a tool for running local Kubernetes clusters using Docker container “nodes”.
kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

## Installation of tools

### Kubectl
Kubectl is the command line tool used to interact with Kuberenetes API. It is one single statically compiled binary, so
it's very easy to install.  Just follow instructions [here](https://kubernetes.io/docs/tasks/tools/#kubectl)

### Container runtime
We can use either docker or podman. Unless you have a license for docker desktop and hence most likely already 
installed; I recommend installing podman

* [podman installation](https://podman.io/)
* [docker installation](https://www.docker.com/)

### Kind

[Kind installation](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

## Setting up the cluster

Once all the tools are installed, its as easy as  running:
```commandline
kind create cluster
```
This will start your cluster and create a kubectl config with the name `kind-kind` and set that as the default.

Once the cluster is up you can run:

```commandline
kubectl get namespaces
```
expected response
```
NAME                 STATUS   AGE
default              Active   117s
kube-node-lease      Active   117s
kube-public          Active   117s
kube-system          Active   117s
local-path-storage   Active   113s
```

This command shows the namespaces in the cluster. To see which containers are running in which namespaces we can run the 
following command:

```commandline
kubectl get pods -A
```
expected response:
```
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-76f75df574-48n62                     1/1     Running   0          4m21s
kube-system          coredns-76f75df574-jht7w                     1/1     Running   0          4m21s
kube-system          etcd-kind-control-plane                      1/1     Running   0          4m36s
kube-system          kindnet-sp4r4                                1/1     Running   0          4m22s
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          4m36s
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          4m36s
kube-system          kube-proxy-dczl4                             1/1     Running   0          4m22s
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          4m37s
local-path-storage   local-path-provisioner-7577fdbbfb-zbrbf      1/1     Running   0          4m21s
```

We can see that there are a bunch of containers running in the kube-system namespace and the local-path-storage
namespace.  These containers provide functionality such as dns, networking, storage among other things.  We will not do
anything with these containers in this lab.   Instead, we will deploy our own containers to the `default` namespace.


## Installing Wordpress

To deploy Wordpress we need the following resources.
* pod for wordpress
* pod for mysql
* persistent storage wordpress and mysql
* services for wordpress and mysql

The  Manifests for this can be found in the `deploy` folder.  

### Storage

In kubernetes there are two different resource types for storage one is called `PersistentVolume` or `pv`  this is the actual
volume containing the data, and there is `PersistentVolumeClaim` or `pvc`.  The `pvc` is a claim for storage and the
recommended way to use persistent volumes in kubernetes. When using the PVC  kubernetes will figure out what physical
volume is associated with the claim and it will also provision that volume when the `pvc` is used for the first time.

To deploy the storage claims run:
```commandline
kubectl apply -f deploy/storage.yaml
```
```
persistentvolumeclaim/mysql-pv-claim created
persistentvolumeclaim/wp-pv-claim created
```

We can now run:
```commandline
kubectl get pvc -o wide
```
```
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE   VOLUMEMODE
mysql-pv-claim   Pending                                      standard       <unset>                 10m   Filesystem
wp-pv-claim      Pending                                      standard       <unset>                 10m   Filesystem
```
We can see that these claims are pending. Since they have not been used yet

If we run:
```commandline
kubectl get pv -o wide
```
```
No resources found
```
Since no actual volumes has been created yet.

### Mysql

We will now deploy the mysql pod. Normally a kubernetes [secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
would be used, but for sake of simplicity in this lab we will just add the secret in plain text to the environment
variables of the pod.

Deploying the mysql database:
```commandline
kubectl apply -f deploy/mysql-deployment.yaml 
```
```commandline
kubectl get pods -o wide
```
```
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
wordpress-mysql-99c96875-5ljjs   1/1     Running   0          53s   10.244.0.6   kind-control-plane   <none>           <none>
```
We can se that the pod is up and running.

Lets check our volumes
```commandline
kubectl get pv,pvc
```
```
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-3348225c-5231-4c39-bc46-251fb3bda21b   1Gi        RWO            Delete           Bound    default/mysql-pv-claim   standard       <unset>                          2m30s

NAME                                   STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/mysql-pv-claim   Bound     pvc-3348225c-5231-4c39-bc46-251fb3bda21b   1Gi        RWO            standard       <unset>                 92m
persistentvolumeclaim/wp-pv-claim      Pending     
```
We can see that the `pv` for mysql has been created and is bound to our pod, while the `pvc` for wordpress has not yet
been created.

#### The service
In order to be able to connect to the database from the webserver we will need a persistent hostname.  This is achieved 
by creating a service. A service can be seen like a simple network load balancer.

```commandline
kubectl apply -f deploy/mysql-service.yaml
```
```
kubectl get svc
NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
kubernetes        ClusterIP   10.96.0.1     <none>        443/TCP    116m
wordpress-mysql   ClusterIP   10.96.66.18   <none>        3306/TCP   8s
```
We can see two services here,  one is the kub

