# Ceph Cluster and Ceph CSI on Kubernetes

I have a home lab running in some Intel NUC where I have a Kubernetes cluster that does almost everything I need (I still need an external DNS... this is in progress), and I needed something to provide dynamic volume provisioning.

So far, I have created a generic storage class and used a no-provisioner to map persistent volumes to empty directories. Still, it only works well in the short run because the volumes are not replicated and not dynamically provisioned and some other problems of empty dir as a PV.

On top of that, I worked with Ceph a few years ago and wanted to remember some stuff, so I merged my need with my will to return to Ceph.

Now, I have a Ceph Cluster + Kubernetes cluster, and I can provision persistent volumes dynamically.

About the External DNS, I have the PDNS already configured, so when I have some time, I will give it a try: https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/pdns.md.

- [Ceph Cluster and Ceph CSI on Kubernetes](#ceph-cluster-and-ceph-csi-on-kubernetes)
  - [Cluster Configuration (VMs)](#cluster-configuration-vms)
  - [Repository setup (Al nodes)](#repository-setup-al-nodes)
  - [Node 1 (ceph1):](#node-1-ceph1)
  - [Configure Kubernetes to use Ceph to provisioning Persistent volumes](#configure-kubernetes-to-use-ceph-to-provisioning-persistent-volumes)
    - [ConfigMap with CSI configuration:](#configmap-with-csi-configuration)
    - [ConfigMap with KMS configuration.](#configmap-with-kms-configuration)
    - [Another config map required for Ceph CSI (I don't remember why):](#another-config-map-required-for-ceph-csi-i-dont-remember-why)
    - [Secret with the user and key created before.](#secret-with-the-user-and-key-created-before)
    - [Create the Roles](#create-the-roles)
    - [Deploy the provisioner and the node plugin](#deploy-the-provisioner-and-the-node-plugin)
    - [Create the StorageClass](#create-the-storageclass)
  - [Testing the configuration](#testing-the-configuration)
  - [For those lazy (I love copy and paste too)](#for-those-lazy-i-love-copy-and-paste-too)
  - [Helm Charts](#helm-charts)
  - [References:](#references)
  - [TODO](#todo)

## Cluster Configuration (VMs)

OS Version:
```bash
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.3 LTS
Release:        22.04
Codename:       jammy
```

CPU(s): `4`
RAM: `4GB`
DISKS:
```bash
/dev/sda  20G -> OS
/dev/sdb  50G -> Ceph OSD
```

## Repository setup (Al nodes)
```bash
curl https://download.ceph.com/keys/release.gpg -o /etc/apt/keyrings/ceph.gpg
apt-add-repository "deb https://download.ceph.com/debian-reef/ $(lsb_release -sc) main"
```
Edit the file `/etc/apt/sources.list.d/ceph.list` and add the `signed-by=/etc/apt/keyrings/ceph.gpg`:
```bash
deb [signed-by=/etc/apt/keyrings/ceph.gpg] https://download.ceph.com/debian-reef/ jammy main
```

Install Podman (used by Cephadm)
```bash
apt update
apt install podman -y
```

## Node 1 (ceph1):
```bash
apt update
apt install cephadm
cephadm bootstrap --mon-ip 192.168.10.56
cephadm add-repo --release reef
cephadm install ceph-common
```
Optional (but why not help the community):
```bash
ceph telemetry on --license sharing-1-0
```
Continue with the cluster configuration:
```bash
ceph status
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2.lab.local
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3.lab.local
ceph status
ceph orch host add ceph2 --labels _admin
ceph orch host add ceph3 --labels _admin
ceph orch daemon add mon ceph2
ceph orch daemon add mon ceph3
ceph orch apply mon --placement="ceph1,ceph2,ceph3"
ceph orch device ls
ceph orch apply osd --all-available-devices
```

Add the Kubernetes pool:

```bash
ceph osd pool create kubernetes
rbd pool init kubernetes
```

Create the user for Kubernetes (save this information, you will need it later):

```bash
ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
[client.kubernetes]
        key = AQClQkZlHuZYLRAA0JuPOkWGtNujPrLwNoNcXQ==
```

Dump the Ceph configuration (take note of FSID and monitors, you will need it later):

```bash
ceph mon dump
epoch 3
fsid 233a71ec-7b10-11ee-a5c8-f76cb62122db
last_changed 2023-11-04T13:00:43.847269+0000
created 2023-11-04T12:46:35.895367+0000
min_mon_release 18 (reef)
election_strategy: 1
0: [v2:192.168.10.56:3300/0,v1:192.168.10.56:6789/0] mon.ceph1
1: [v2:192.168.10.58:3300/0,v1:192.168.10.58:6789/0] mon.ceph2
2: [v2:192.168.10.59:3300/0,v1:192.168.10.59:6789/0] mon.ceph3
dumped monmap epoch 3
```

In case the `ceph health` or `ceph status` command return a WARN because OSD performance, you can disable the scrub and deep-scrub and re-enable it after some time:

```bash
# ceph osd set nodeep-scrub
# ceph osd set nodeep-scrub

# ceph osd unset noscrub
# ceph osd unset nodeep-scrub
```

## Configure Kubernetes to use Ceph to provisioning Persistent volumes

### ConfigMap with CSI configuration:

```bash
cat <<EOF > csi-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "233a71ec-7b10-11ee-a5c8-f76cb62122db",  #FSID (remember?)
        "monitors": [
          "192.168.10.56:6789",
          "192.168.10.58:6789",
          "192.168.10.59:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
EOF

kubectl create -f csi-config-map.yaml
```

### ConfigMap with KMS configuration. 

It's mandatory even if you don't use KMS. If you don't use KMS, just leave the config.json empty (my case):
```bash
cat <<EOF > csi-kms-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
EOF

kubectl create -f csi-kms-config-map.yaml
```

### Another config map required for Ceph CSI (I don't remember why):
```bash
cat <<EOF > ceph-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  ceph.conf: |
    [global]
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
  # keyring is a required key and its value should be empty
  keyring: |
metadata:
  name: ceph-config
EOF

kubectl create -f ceph-config-map.yaml
```

### Secret with the user and key created before. 

This will be used to create the images/blocks in the pool that we created before:
```bash
cat <<EOF > csi-rbd-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  userID: kubernetes
  userKey: AQClQkZlHuZYLRAA0JuPOkWGtNujPrLwNoNcXQ== #key (remember?)
EOF

kubectl create -f csi-rbd-secret.yaml
```

###  Create the Roles
We need to let Ceph CSI work, so let's give it the permissions that it needs:
```bash
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
```

### Deploy the provisioner and the node plugin
Before apply, edit the yaml before and fix the pod applyingn and replace `quay.io/cephcsi/cephcsi:canary` with `quay.io/cephcsi/cephcsi:v3.8.1`.

```bash
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml
kubectl apply -f csi-rbdplugin-provisioner.yaml
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin.yaml
kubectl apply -f csi-rbdplugin.yaml
```

### Create the StorageClass
```bash
cat <<EOF > csi-rbd-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: 233a71ec-7b10-11ee-a5c8-f76cb62122db #FSID (remember?)
   pool: kubernetes
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
EOF
kubectl apply -f csi-rbd-sc.yaml
```

Once this first part is done, you can use the new storage class to provision persistent volumes directly from Ceph.
You may want to create a new namespace for this configuration so you can delete it later and don't mess with your current environment.

## Testing the configuration 

Some tests validate the whole configuration (I assume you're familiar with Kubernetes, and you know how to check if the PVC is bound or not). Help yourself after each PVC is created.

Create a PVC to be used as raw block device (`volumeMode: Block`):
```bash
cat <<EOF > raw-block-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
EOF
kubectl apply -f raw-block-pvc.yaml
```

Create a pod with the PVC as a raw block device:
```bash
cat <<EOF > raw-block-pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-raw-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: ["tail -f /dev/null"]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: raw-block-pvc
EOF
kubectl apply -f raw-block-pod.yaml
```

Create a new PVC now to be used as FS (`volumeMode: Filesystem`):
```bash
cat <<EOF > pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
EOF
kubectl apply -f pvc.yaml
```

Create a pod and mount the PVC as a FS:
```bash
cat <<EOF > pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: csi-rbd-demo-pod
spec:
  containers:
    - name: web-server
      image: nginx
      volumeMounts:
        - name: mypvc
          mountPath: /var/lib/www/html
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: rbd-pvc
        readOnly: false
EOF
kubectl apply -f pod.yaml
```

## For those lazy (I love copy and paste too)

All files used to configure the CSI are in this repo, so make sure to gather all information (kubernetes client key, fsid, mons...), fix in the files and apply in the following order:

```bash 
kubectl create -f csi-config-map.yaml
kubectl create -f csi-kms-config-map.yaml
kubectl create -f ceph-config-map.yaml
kubectl create -f csi-rbd-secret.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
kubectl apply -f csi-rbd-sc.yaml
kubectl apply -f raw-block-pvc.yaml
kubectl apply -f raw-block-pod.yaml
kubectl apply -f pvc.yaml
kubectl apply -f pod.yaml
```

## Helm Charts

Add the Ceph CSI Helm Charts Repo

```bash
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm repo update
```

The file `values.yaml` does everything that all the other files to install and configure the Ceph CSI above would do.

So to acomplish the same result, you can simply run the following command:

```bash
helm install --namespace "ceph-csi-rbd" --create-namespace "ceph-csi-rbd" ceph-csi/ceph-csi-rbd -f values.yaml
```

The command above will organize everything in the namespace `ceph-csi-rbd` and set the `csi-rbd-sc` as default storage class in your cluster.

If you don't want the Ceph as default storage class, comment the following configuration in `values.yaml`

```
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
```

To test you can run one of the previous tests like `kubectl apply -f pvc.yaml`, below an example:

```bash
$ kubectl apply pvc.y^C
$ kubectl get pv
No resources found
$ kubectl get pvc
No resources found in default namespace.
$ kubectl apply -f pvc.yaml
persistentvolumeclaim/rbd-pvc created
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
pvc-c4e16a77-3801-4861-aee1-d93bac9ca393   1Gi        RWO            Delete           Bound    default/rbd-pvc   csi-rbd-sc              2s
$ kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rbd-pvc   Bound    pvc-c4e16a77-3801-4861-aee1-d93bac9ca393   1Gi        RWO            csi-rbd-sc     5s
$ kubectl delete -f pvc.yaml
persistentvolumeclaim "rbd-pvc" deleted
```

## References:

https://docs.ceph.com/en/reef/cephadm/install/
https://docs.ceph.com/en/reef/cephadm/host-management/#cephadm-adding-hosts
https://docs.ceph.com/en/reef/cephadm/services/osd/#cephadm-deploy-osds
https://docs.ceph.com/en/reef/rbd/rbd-kubernetes/
https://github.com/ceph/ceph-csi/tree/devel/charts/ceph-csi-rbd
https://github.com/ceph/ceph-csi/tree/devel/charts/ceph-csi-cephfs

## TODO

- [x] Make all the Kubernetes part of this via Helm Charts for the sake of our menthal health.