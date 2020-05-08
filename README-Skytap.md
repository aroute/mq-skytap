### New Project
```
oc new-project mqtest
```
### Git clone
```
git clone https://github.com/IBM/charts.git
git checkout stable/ibm-mqadvanced-server-dev
cp -r /root/oms-helmchart/charts/stable/ibm-mqadvanced-server-dev/ .
```
### SCC
```
cd ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration
./createSecurityClusterPrereqs.sh

cd ibm_cloud_pak/pak_extensions/pre-install/namespaceAdministration
./createSecurityNamespacePrereqs.sh mqtest
```
### Password
```
echo -n password | base64
```
### Secret
```
cat <<EOF | oc create -f -
apiVersion: v1
kind: Secret
metadata:  
    name: mq-secret
type: Opaque
data:  
    adminPassword: cGFzc3dvcmQ=
    appPassword: cGFzc3dvcmQ=
EOF
```
### PV
```
cat <<EOF | oc create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mq-data
  labels:
    app: ibm-mq
spec:
  capacity:
    storage: 2Gi 
  accessModes:
    - ReadWriteOnce 
  persistentVolumeReclaimPolicy: Retain 
  nfs: 
    path: /var/nfs-data/
    server: 172.16.1.1
    readOnly: false

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mq-log
  labels:
    app: ibm-mq
spec:
  capacity:
    storage: 2Gi 
  accessModes:
    - ReadWriteMany 
  persistentVolumeReclaimPolicy: Retain 
  nfs: 
    path: /var/nfs-data/
    server: 172.16.1.1
    readOnly: false

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mq-qm
  labels:
    app: ibm-mq
spec:
  capacity:
    storage: 2Gi 
  accessModes:
    - ReadWriteMany 
  persistentVolumeReclaimPolicy: Retain 
  nfs: 
    path: /var/nfs-data/
    server: 172.16.1.1
    readOnly: false
EOF
```
### Permission fix
```
chmod -R 777 nfs-data/
```
### Helm
```
helm repo add ibm-stable-charts https://raw.githubusercontent.com/IBM/charts/master/repo/stable 

helm install --name mqtest ibm-stable-charts/ibm-mqadvanced-server-dev -f override.yaml

helm install --name mqtest ibm-stable-charts/ibm-mqadvanced-server-dev --set license=accept --set queueManager.dev.secret.name=mq-secret --set queueManager.dev.secret.adminPasswordKey=adminPassword --set persistence.useDynamicProvisioning=false
```

### Helm delete
```
helm delete mqtest --purge
oc patch pv/data --type json -p $'- op: remove\n  path: /spec/claimRef'

Get the MQ Console URL by running these commands:
  export CONSOLE_ROUTE=$(kubectl get route mqtest-ibm-mq-web -n mqtest -o jsonpath="{.spec.host}")
  echo https://$CONSOLE_ROUTE/ibmmq/console

The MQ connection information for clients inside the cluster is as follows:
  mqtest-ibm-mq.mqtest.svc:1414

To get your 'admin' user password run:
    MQ_ADMIN_PASSWORD=$(kubectl get secret --namespace mqtest mq-secret -o jsonpath="{.data.password}" | base64 --decode; echo)
```
