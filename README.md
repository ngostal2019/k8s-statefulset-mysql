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
> Finally, update the new storage class to be used by default for PersistentVolumeClaim
```sh
kubectl patch storageclass csi-hostpath-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
## Apply the Kubernetes Manifest Files
> Folder structure
```sh
├── mysql-configmap.yaml
├── mysql-stastefulset.yaml
├── mysql-svc.yaml
└── README.md
---
kubectl apply -f mysql-configmap.yaml -f mysql-svc.yaml -f mysql-stastefulset.yaml
```
- Check the pods
```sh
kubectl get pods

output:
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   2/2     Running   0          3m
mysql-1   2/2     Running   0          2m
mysql-2   2/2     Running   0          2m
```
- Check the statefulset
```sh
kubectl get sts

output:
NAME    READY   AGE
mysql   3/3     4m
```
- Check the configMap
```sh
kubectl get cm

output:
NAME               DATA   AGE
kube-root-ca.crt   1      5m
mysql              2      1m
```
- Check the services
```sh
kubectl get svc

output:
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    5m
mysql        ClusterIP   None            <none>        3306/TCP   1m
mysql-read   ClusterIP   10.99.185.183   <none>        3306/TCP   1m
```
> Try writing to the database
```sh
kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('Hello there, Staline ngoma is testing Mysql on Kubernetes');
EOF
```