*TODO: Files with sensitive data; move to Vault*
```
# line 6-8: services/Guacamole/configMap_Guacamole.yml
```

# Kubernetes.K3s.installLog
*3 VM's provisioned with Ubuntu Server 18.04*

## K3s cluster 
On first node:
```  
curl -sfL https://get.k3s.io | sh -s - --no-deploy traefik
cat /var/lib/rancher/k3s/server/token
kubectl config view --raw
```
On subsequent nodes:
```
curl -sfL https://get.k3s.io | K3S_URL=https://<fqdn or ip>:6443 K3S_TOKEN=<value from master> sh -
```

### 0) Configure automatic updates
Install Rancher's [System Upgrade Controller](https://rancher.com/docs/k3s/latest/en/upgrades/automated/):
```
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.6.2/system-upgrade-controller.yaml
```
Apply a [server (master node)](https://code.spamasaurus.com/djpbessems/Kubernetes.K3s.installLog/src/branch/master/system/UpgradeController/plan-Server.yml) and [agent (worker node)](https://code.spamasaurus.com/djpbessems/Kubernetes.K3s.installLog/src/branch/master/system/UpgradeController/plan-Agent.yml) plan:
```
kubectl apply -f system/UpgradeController/plan-Server.yml -f system/UpgradeController/plan-Agent.yml
```

### 1) Persistent storage

#### 1.1) `storageClass` for SMB (CIFS):
See https://github.com/kubernetes-csi/csi-driver-smb:
```
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/deploy/install-driver.sh | bash -s master --
```
Store credentials in `secret`:
```
kubectl create secret generic smb-credentials --from-literal username=<<omitted>> --from-literal domain=<<omitted>> --from-literal password=<<omitted>>
```

#### 1.2) `flexVolume` for SMB (CIFS):
```
curl -Ls https://github.com/juliohm1978/kubernetes-cifs-volumedriver/blob/master/install.yaml -o storage/flexVolSMB/daemonSet-flexVolSMB.yml
```
Override drivername to something more sensible (see [storage/flexVolSMB/daemonSet-flexVolSMB.yml](https://code.spamasaurus.com/djpbessems/Kubernetes.K3s.installLog/src/branch/master/storage/flexVolSMB/daemonSet-flexVolSMB.yml))
```
spec:
  template:
    spec:
      containers:
        - image: juliohm/kubernetes-cifs-volumedriver-installer:2.0
          ...
          env:
            - name: VENDOR
              value: mount
            - name: DRIVER
              value: smb
          ...
```
Perform installation:
```
kubectl apply -f storage/flexVolSMB/daemonSet-flexVolSMB.yml
```
Wait for installation to complete (check logs of all installer-pods), then delete `daemonSet`:
```
kubectl delete -f storage/flexVolSMB/daemonSet-flexVolSMB.yml
```
Store credentials in `secret`:
```
kubectl create secret generic --type=mount/smb smb-secret --from-literal=username=<<omitted>> --from-literal=password=<<omitted>>
```

#### 1.3) `storageClass` for distributed block storage:
See [Longhorn Helm Chart](https://longhorn.io/):
```
kubectl create namespace longhorn-system
helm repo add longhorn https://charts.longhorn.io
helm install longhorn longhorn/longhorn --namespace longhorn-system --values=storage/Longhorn/chart-values.yml
```
Expose Longhorn's dashboard through  `IngressRoute`:
```
kubectl apply -f storage/Longhorn/ingressRoute-Longhorn.yml
```
Add additional `storageClass` with backup schedule:  
***After** specifying a NFS backup target (syntax: `nfs://servername:/path/to/share`) through Longhorn's dashboard*
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

### 2) Ingress Controller
##### 2.1) Create `configMap`, `secret` and `persistentVolumeClaim`
The `configMap` contains Traefik's static and dynamic config:
```
kubectl apply -f ingress/Traefik2.x/configMap-Traefik.yml
```

The `secret` contains credentials for Cloudflare's API:
```
kubectl create secret generic traefik-cloudflare --from-literal=CF_API_EMAIL=<<omitted>> --from-literal=CF_API_KEY=<<omitted>> --namespace kube-system
```

The `persistentVolumeClaim` will contain `/data/acme.json` (referenced as `existingClaim`):
```
kubectl apply -f ingress/Traefik2.x/persistentVolumeClaim-Traefik.yml
```
##### 2.2) Install Helm Chart
See [Traefik 2.x Helm Chart](https://github.com/containous/traefik-helm-chart):
```
helm repo add traefik https://containous.github.io/traefik-helm-chart
helm repo update
helm install traefik traefik/traefik --namespace kube-system --values=ingress/Traefik2.x/chart-values.yml
```
##### 2.3) Replace `IngressRoute` for Traefik's dashboard:
```
kubectl apply -f ingress/Traefik2.x/ingressRoute-Traefik.yaml
kubectl delete ingressroute traefik-dashboard --namespace kube-system
```

### 3) Secret management
*Perform these steps **after** configuring persistent storage **and** ingress*  
##### 3.1) Create `persistentVolume` and `ingressRoute`
*Requires specifying a `uid` & `gid` in the flexvolSMB-`persistentVolume`*  
```
kubectl create namespace vault
kubectl apply -f services/Vault/persistentVolume-Vault.yml
kubectl apply -f services/Vault/ingressRoute-Vault.yml
```
##### 3.2) Install Helm Chart
See [HashiCorp Vault](https://www.vaultproject.io/docs/platform/k8s/helm/run):  
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault --namespace vault --values=services/Vault/chart-values.yml
```
Configure Vault for use;
- Enable Kubernetes authentication (see https://www.vaultproject.io/api-docs/auth/kubernetes)
- Store basic access policy template
- Enable `kv`-engine
```
# kubectl exec -n vault -it vault-0 -- sh

# It might be necessary to first login with an existing token:
# vault login
vault auth enable kubernetes
vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

cat <<EOF > /home/vault/app-policy.hcl
path "secret*" {
  capabilities = ["read"]
}
EOF

vault secrets enable -path=secret -version=2 kv
```
### 4) Services
##### 4.1) [Adminer](https://www.adminer.org/)    <small>(SQL management)</small>
```
kubectl apply -f services/Adminer/configMap-Adminer.yml
kubectl apply -f services/Adminer/deploy-Adminer.yml
```
Vault configuration:
```
vault kv put secret/adminer \
  sqlitepw=<value>
vault write auth/kubernetes/role/adminer \
  bound_service_account_names=adminer \
  bound_service_account_namespaces=default \
  policies=adminer \
  ttl=1h
vault policy write adminer /home/vault/app-policy.hcl
```
##### 4.2) [Bitwarden_rs](https://github.com/dani-garcia/bitwarden_rs)    <small>(password manager)</small>
*Requires [mount.cifs](https://linux.die.net/man/8/mount.cifs)' option `nobrl`*
```
kubectl apply -f services/Bitwarden/deploy-Bitwarden.yml
```
Vault configuration:
```
vault kv put secret/bitwarden \
  admintoken=<value> \
  yubicoclientid=<value> \
  yubicosecretkey=<value>
vault write auth/kubernetes/role/bitwarden \
  bound_service_account_names=bitwarden \
  bound_service_account_namespaces=default \
  policies=bitwarden \
  ttl=1h
vault policy write bitwarden /home/vault/app-policy.hcl
```
##### 4.3) [DroneCI](https://drone.io/)    <small>(contineous delivery)</small>
```
kubectl apply -f services/DroneCI/deploy-DroneCI.yml
```
Vault configuration:
```
vault kv put secret/drone \
  rpcsecret=<value> \
  giteaclientid=<value> \
  giteaclientsecret=<value>
vault write auth/kubernetes/role/drone \
  bound_service_account_names=drone \
  bound_service_account_namespaces=default \
  policies=drone \
  ttl=1h
vault policy write drone /home/vault/app-policy.hcl
```
##### 4.4) [Gitea](https://gitea.io/)    <small>(git repository)</small>
```
kubectl apply -f services/Gitea/deploy-Gitea.yml
```
##### 4.5) [Gotify](https://gotify.net/)    <small>(notifications)</small>
```
kubectl apply -f services/Gotify/deploy-Gotify.yml
```
##### 4.6) [Guacamole](https://guacamole.apache.org/doc/gug/guacamole-docker.html)    <small>(remote desktop gateway)</small>
*Requires specifying a `uid` & `gid` in both the `securityContext` of the MySQL container and the `persistentVolume`*
```
kubectl apply -f services/Guacamole/configMap-Guacamole.yml
kubectl apply -f services/Guacamole/deploy-Guacamole.yml
```
Wait for the included containers to start, then perform the following commands to initialize the database:
```
kubectl exec -i guacamole-<pod-id> --container guacamole -- /opt/guacamole/bin/initdb.sh --mysql > initdb.sql
kubectl exec -i guacamole-<pod-id> --container mysql -- mysql -uguacamole -pguacamole guacamole < initdb.sql
kubectl rollout restart deployment guacamole
```
##### 4.7) [Harbor](https://goharbor.io/)    <small>(container image registry)</small>
Create `ingressRoute` and `storageClass`
```
kubectl create namespace harbor
kubectl apply -f services/Harbor/ingressRoute-Harbor.yml
kubectl apply -f services/Harbor/storageClass-Harbor.yml
```
Install Helm chart
```
helm repo add harbor https://helm.goharbor.io
helm repo update
helm install harbor harbor/harbor --namespace harbor --values=services/Harbor/chart-values.yml
```

##### 4.8) [Lighttpd](https://www.lighttpd.net/)    <small>(webserver)</small>
*Serves various semi-containerized websites; respective webcontent is stored on fileshare*  
```
kubectl apply -f services/Lighttpd/configMap-Lighttpd.yml
kubectl apply -f services/Lighttpd/deploy-Lighttpd.yml
kubectl apply -f services/Lighttpd/cronJob-Spotweb.yml
```
##### 4.9) PVR `namespace`    <small>(automated media management)</small>
*Containers use shared resources to be able to interact with downloaded files*
```
kubectl create secret generic --type=mount/smb smb-secret --from-literal=username=<<omitted>> --from-literal=password=<<omitted>> -n pvr
kubectl apply -f services/PVR/persistentVolumeClaim-PVR.yml
kubectl apply -f services/PVR/storageClass-PVR.yml
```
###### 4.9.1) [NZBHydra](https://github.com/theotherp/nzbhydra2)    <small>(index aggregator)</small>
```
kubectl apply -f services/PVR/deploy-NZBHydra.yml
```
###### 4.9.2) [Plex](https://www.plex.tv/)    <small>(media library)</small>
*Due to usage of symlinks, partially incompatible with SMB-share-backed storage*
```
kubectl apply -f services/PVR/deploy-Plex.yml
```
After deploying, Plex server needs to be *claimed* (=assigned to Plex-account):
```
kubectl get endpoints Plex -n PVR
```
Browse to the respective IP address (http://<nodeipaddress>:32400/web) and follow instructions.
###### 4.9.3) [Radarr](https://radarr.video/)    <small>(movie management)</small>
```
kubectl apply -f services/PVR/deploy-Radarr.yml
```
###### 4.9.4) [Readarr](https://readarr.com/)    <small>(book management)</small>
```
kubectl apply -f services/PVR/deploy-Readarr.yml
```
###### 4.9.5) [SABnzbd](https://sabnzbd.org/)    <small>(download client)</small>
```
kubectl apply -f services/PVR/deploy-SABnzbd.yml
```
###### 4.9.6) [Sonarr](https://sonarr.tv/)    <small>(tv management)</small>
```
kubectl apply -f services/PVR/deploy-Sonarr.yml
```

##### 4.10) [Shaarli](https://github.com/shaarli/Shaarli)    <small>(bookmarks/notes)</small>
```
kubectl apply -f services/Shaarli/deploy-Shaarli.yml
```
##### 4.11) [Theia](https://theia-ide.org/)    <small>(web IDE)</small>
```
kubectl apply -f services/Theia/deploy-Theia.yml
```
##### 4.12) [Traefik-Certs-Dumper](https://github.com/ldez/traefik-certs-dumper)    <small>(certificate tooling)</small>
```
kubectl apply -f services/TraefikCertsDumper/deploy-TraefikCertsDumper.yml
```
### 5) Miscellaneous
*Various notes/useful links*  

* Replacement for [not-yet-deprecated](https://github.com/kubernetes/kubectl/issues/151) `kubectl get all -A`:

      
      kubectl get $(kubectl api-resources --verbs=list -o name | paste -sd, -) --ignore-not-found --all-namespaces
* `DaemonSet` to configure nodes' **sysctl** `fs.inotify.max-user-watches`:

      
      kubectl apply -f system/InotifyMaxWatchers/daemonSet-InotifyMaxWatchers.yml
