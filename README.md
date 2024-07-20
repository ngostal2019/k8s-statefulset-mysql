# Create a StatefulSet Mysql Cluster with 1 Primary and 2 Readers
> [!NOTE] 
> It is advised to have a cluster with at least 2 nodes but in our case, we will spin up a K8S cluster with 3 nodes
```sh
minikube start -3
```
> [!WARNING]
> Before deploying the resources, you'll need to remove/add a few minikube addons for dynamic persistent volume provisionning
```sh
minikube addons list
minikube addons disable storage-provisioner
minikube addons disable default-storageclass
minikube addons enable volumesnapshots
minikube addons enable csi-hostpath-driver # This may take 5 to 10 min
```
> Finally, update the new storage class to be use by default
```sh
kubectl patch storageclass csi-hostpath-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```