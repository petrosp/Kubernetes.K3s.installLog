### Persistent Storage
Manifest for [Longhorn](https://github.com/longhorn/longhorn):
```
curl -Ls https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml -o storage/Longhorn/deploy-Longhorn.yaml
sed -e 's/LoadBalancer/ClusterIP/' -i storage/Longhorn/deploy-Longhorn.yaml
kubectl apply -f storage/Longhorn/deploy-Longhorn.yaml
```
##### `IngressRoute` for Longhorn's dashboard:
```
kubectl apply -f storage/Longhorn/ingressRoute-Longhorn.yaml
```
##### `storageClass` with backup schedule:
After specifying a NFS backup target (syntax: `nfs://servername:/path/to/share`) through Longhorn's dashboard, create a new `storageClass` with backup schedule:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn-dailybackup
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  recurringJobs: '[{"name":"backup", "task":"backup", "cron":"0 0 * * *", "retain":14}]'
```
Then make this the new default `storageClass`:
```
kubectl patch storageclass longhorn-dailybackup -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl delete storageclass longhorn
```