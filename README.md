# Create a StatefulSet Mysql Cluster with 1 Primary and 2 Readers
> [!NOTE] 
> It is advised to have a cluster with at least 2 nodes but in our case, we will spin up a K8S cluster with 3 nodes
```sh
minikube start -n 3
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
> Try writing to the primary mysql server
```sh
kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('Hello there, Staline ngoma is testing Mysql on Kubernetes');
EOF
```
> Use the hostname mysql-read to send test queries to any server that reports being Ready:
```sh
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-read -e "SELECT * FROM test.messages"

output:
+-----------------------------------------------------------+
| message                                                   |
+-----------------------------------------------------------+
| Hello there, Staline ngoma is testing Mysql on Kubernetes |
+-----------------------------------------------------------+
pod "mysql-client" deleted
```
> To demonstrate that the mysql-read Service distributes connections across servers, you can run SELECT @@server_id in a loop:
```sh
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
```
## Simulate Pod and Node failure
> Break the Readiness probe
```sh
kubectl exec mysql-2 -c mysql -- mv /usr/bin/mysql /usr/bin/mysql.off
```
> Delete the pods so it can be recreated by the sts
```sh
kubectl delete pod mysql-2
```
> Drain the nod to mark it unschedulable
```sh
kubectl get pod mysql-2 -o wide
kubectl drain <node-name> --force --delete-emptydir-data --ignore-daemonsets
kubectl get pod mysql-2 -o wide --watch
kubectl uncordon <node-name> # To mark it schedulable
```
> Scale the number of replicas and try to read from the new replicas
```sh
kubectl scale statefulset mysql  --replicas=5
kubectl get pods -l app=mysql --watch
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-3.mysql -e "SELECT * FROM test.messages"
  kubectl scale statefulset mysql --replicas=3 # To scale down
```
> [!Note]
> Although scaling up creates new PersistentVolumeClaims automatically, scaling down does not automatically delete these PVCs.
```sh
kubectl get pvc -l app=mysql
# To delete them
kubectl delete pvc data-mysql-3
kubectl delete pvc data-mysql-4
```
## Cleaning up
1. Cancel the SELECT @@server_id loop by pressing Ctrl+C in its terminal, or running the following from another terminal:
```sh
kubectl delete pod mysql-client-loop --now
```
2. Delete the StatefulSet. This also begins terminating the Pods.
```sh
kubectl delete statefulset mysql
```
3. Verify that the Pods disappear. They might take some time to finish terminating.
```sh
kubectl get pods -l app=mysql
```
4. Delete the ConfigMap, Services, and PersistentVolumeClaims.
```sh
kubectl delete configmap,service,pvc -l app=mysql
```