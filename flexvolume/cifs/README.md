# CIFS/SMB flex volume driver for Kubernetes (Preview)
 - supported Kubernetes version: v1.8, v1.9
 - supported agent OS: Linux 

# Install cifs flex volume driver on a kubernetes cluster
## 1. config kubelet service (skip this step in [AKS](https://azure.microsoft.com/en-us/services/container-service/) or from [acs-engine](https://github.com/Azure/acs-engine) v0.12.0)
specify `volume-plugin-dir` in kubelet service config 
```
sudo vi /etc/systemd/system/kubelet.service
  --volume=/etc/kubernetes/volumeplugins:/etc/kubernetes/volumeplugins:rw \
        --volume-plugin-dir=/etc/kubernetes/volumeplugins \
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Note:
 - `/etc/kubernetes/volumeplugins` has already been the default flexvolume plugin directory in acs-engine (starting from v0.12.0)
 - There would be one line of [kubelet log](https://github.com/andyzhangx/Demo/tree/master/debug#q-how-to-get-k8s-kubelet-logs-on-linux-agent) in agent node like below showing that `flexvolume-azure/cifs` is loaded correctly
```
I0122 08:24:47.761479    2963 plugins.go:469] Loaded volume plugin "flexvolume-azure/cifs"
```
 - Flexvolume is GA from Kubernetes **1.8** release, v1.7 is depreciated since it does not support [Dynamic Plugin Discovery](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md#dynamic-plugin-discovery).
 
## 2. install cifs flex volume driver on every agent node
### Option#1. Automatically install by k8s daemonset
create daemonset to install cifs driver
```
kubectl create -f https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/cifs/deployment/cifs-flexvol-installer.yaml
```
 - check daemonset status:
```
kubectl describe daemonset cifs-flexvol-installer --namespace=flex
kubectl get po --namespace=flex
```

### Option#2. Manually install on every agent node (depreciated)
```
sudo mkdir -p /etc/kubernetes/volumeplugins/azure~cifs/

cd /etc/kubernetes/volumeplugins/azure~cifs
sudo wget -O cifs https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/cifs/cifs
sudo chmod a+x cifs
```

# Basic Usage
## 1. create a secret which stores cifs account name and password
```
kubectl create secret generic cifscreds --from-literal username=USERNAME --from-literal password="PASSWORD" --type="azure/cifs"
```

## 2. create a pod with cifs flexvolume mount on linux
#### Option#1 Use flexvolume mount directly inside a pod
- download `nginx-flex-cifs.yaml` file and modify `source` field
```
wget -O nginx-flex-cifs.yaml https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/cifs/nginx-flex-cifs.yaml
vi nginx-flex-cifs.yaml
```
 - create a pod with cifs flexvolume driver mount
```
kubectl create -f nginx-flex-cifs.yaml
```

#### Option#2 Create cifs flexvolume PV & PVC and then create a pod based on PVC
 - download `pv-cifs-flexvol.yaml` file, modify `source` field and create a cifs flexvolume persistent volume(PV)
```
wget https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/cifs/pv-cifs-flexvol.yaml
vi pv-cifs-flexvol.yaml
kubectl create -f pv-cifs-flexvol.yaml
```

 - create a cifs flexvolume persistent volume claim(PVC)
```
 kubectl create -f https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/cifs/pvc-cifs-flexvol.yaml
```

 - check status of PV & PVC until its Status changed from `Pending` to `Bound`
 ```
 kubectl get pv
 kubectl get pvc
 ```
 
 - create a pod with cifs flexvolume PVC
```
 kubectl create -f https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/cifs/nginx-flex-cifs-pvc.yaml
 ```

 - watch the status of pod until its Status changed from `Pending` to `Running`
```
watch kubectl describe po nginx-flex-cifs
```

## 3. enter the pod container to do validation
kubectl exec -it nginx-flex-cifs -- bash

```
root@nginx-flex-cifs:/# df -h
Filesystem                                 Size  Used Avail Use% Mounted on
overlay                                    291G  3.2G  288G   2% /
tmpfs                                      3.4G     0  3.4G   0% /dev
tmpfs                                      3.4G     0  3.4G   0% /sys/fs/cgroup
//xiazhang3.file.core.windows.net/k8stest   25G   64K   25G   1% /data
/dev/sda1                                  291G  3.2G  288G   2% /etc/hosts
shm                                         64M     0   64M   0% /dev/shm
tmpfs                                      3.4G   12K  3.4G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                      3.4G     0  3.4G   0% /sys/firmware
```
In the above example, there is a `/data` directory mounted as cifs filesystem.

### Links
[CIFS/SMB wiki](https://en.wikipedia.org/wiki/Server_Message_Block)

[Flexvolume doc](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md)

More clear steps about flexvolume by Redhat doc: [Persistent Storage Using FlexVolume Plug-ins](https://docs.openshift.org/latest/install_config/persistent_storage/persistent_storage_flex_volume.html)
