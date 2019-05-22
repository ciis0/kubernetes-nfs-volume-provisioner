# Kubernetes NFS Volume Provisioner

## UNDER CONSTRUCTION - TODO

* Full documentation

## Introduction

A somewhat flexible Kubernetes controller that can provision NFS Persistent Volumes in a consistent and predictable way. It relies on the `kubectl` binary as a robust API client to watch for PVC events and react in order to provision Persistent Volumes accordingly.

## Installation

Before you can use this controller, you should note a few things.

### Requirements

This controller **does not** provide an NFS server to your cluster. You will need at least one NFS service accessible in your network, and this controller will not give you that.

### Install using kubectl

If you wish to quickly install using `kubectl`, you can use the files included in the `installation` directory. The easiest way is to call the `install` or `uninstall` targets in the `Makefile` provided. This will create all objects in the `default` namespace, and provides a way to get the controller running for a quick test.

```shell
# install
make install

# remove
make unisntall
```

### Install using helm chart

You can also use a helm chart created for this project. The chart is hosted in [one of my Github repositories](https://github.com/juliohm1978/charts). First, add the repo to your local helm installation.

```shell
helm repo add juliohm1978 https://raw.githubusercontent.com/juliohm1978/charts/master/index
helm repo up
```

Now, use the chart `juliohm1978/k8s-nfs-provisioner`.

```shell
helm upgrade --install nfsprov juliohm1978/k8s-nfs-provisioner
```

Uninstall should be a straight forward `helm del`.

```shell
helm del --purge nfsprov
```

To further customize your deployment, take a look at the chart [`values.yaml`](https://github.com/juliohm1978/charts/tree/master/charts/k8s-nfs-provisioner) for details.

## How to use it

Once installed, you will be able to create `StorageClass` objects in you cluster, which provide a number of values on how NFS volumes will be provisioned. You can declare as many StorageClasses as you want with different parameters. All they need in common the field `provisioner: nfs-provisioner.juliohm.com.br`.

Here's a full example:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc01
provisioner: nfs-provisioner.juliohm.com.br
parameters:
  pvNamePrefix: ""
  server: 10.10.10.3
  share: /myshare
  path: /subpath
  namespace: some-namespace
  keepPv: "false"
reclaimPolicy: Delete
```

Now, with `sc01` defined, you can create PersistentVolumeClaims using that use your StorageClass.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: sc01
```

The controller will pick up on that claim and create the corresponding PersistentVolume, which would look similar to:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  labels:
    nfs-provisioner.juliohm.com.br/pvcName: myclaim
    nfs-provisioner.juliohm.com.br/pvcNamespace: default
    nfs-provisioner.juliohm.com.br/storageClassName: sc01
  name: default-myclaim
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 8Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: myclaim
    namespace: default
    uid: 54aff2d6-7a84-11e9-8f64-641c678d3d92
  nfs:
    path: /myshare/subpath/default-myclaim
    server: 10.10.10.3
  persistentVolumeReclaimPolicy: Delete
  storageClassName: sc01
  volumeMode: Filesystem
status:
  phase: Bound
```

Metadata from the StorageClass and PVC are used to create the PV.

> **IMPORTANT**: The PV will contain some special labels related to the StorageClass and PVC used to create it. You should avoid removing or modifying these labels, since they help the controller find PVs that need to be removed from the cluster when their PVC counterparts are no longer available.

The StorageClass parameters may be quite self explanatory, but here is a rundown of what each one means:

```yaml
parameters:
  ## A prefix that should be placed on all PV names created from this StorageClass.
  ## Optional
  ## Default: empty
  pvNamePrefix: ""

  ## Remote NFS server
  ## Required
  server: 10.10.10.3

  ## Share name on the remote NFS server
  ## Required
  share: /myshare

  ## A subdirectory inside the share
  ## Optional
  ## Default: /
  path: /subpath

  ## Namespace to which this StorageClass should be restricted to.
  ## The controller will react to PVC events only from this namespace.
  ## Optional
  ## Default: empty (meaning, react to all PVC events in the cluster)
  namespace: some-namespace

  ## If "true", the PV object will not be deleted when the PVC is deleted.
  ## NOTE: Reclaim policies from this StorageClass will be applied to all PVs
  ## removed from the cluster: REATAIN or DELETE.
  ## Optional
  ## Default: false
  keepPv: "false"
```

The controller uses the StorageClass parameters and the PVC metadata to fully provision the NFS volume. It is possible to use several PersistentVolumes on the same remote NFS share without conflict. To keep storage access unique, a PV will point exactly to:

```text
<REMOTE_SERVER> : <SHARE> + [PATH] + <PVC_NAMESPACE-PVC_NAME>
```

In the above example, the PV provisioned points to:

```text
10.10.10.3:/myshare/subpath/default-myclaim
```

As expected by Kubernetes, the `/subpath/default-myclaim` subdirectories **SHOULD** exist in the remote NFS share. Otherwise, the volume mount inside a Pod will fail.

To make your life easier, the controller allows further configuration as annotations in the PVC. This configuration provides enough information so the controller can automatically create these remote subdirectories before delivering the PV.

Take the following PVC declaration, for example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
  annotations:
    nvs-provisioner.juliohm.com.br/init-perms: "true"
    nvs-provisioner.juliohm.com.br/uid: "1000"
    nvs-provisioner.juliohm.com.br/gid: "1000"
    nvs-provisioner.juliohm.com.br/mode: "644"
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: sc01
```

When the `init-perms` annotation is `true`, the controller will attempt to mount the NFS share temporarily and create the subdirectories `/subpath/default-myclaim`. It will use the values from `uid`, `gid` and `mode` to adjust directory owner and permissions. If this initialization step fails, the PV will not be created.

These annotations allow PVs to be fully provisioned, making sure its volume directories exist on the remote NFS server with the correct owner and permissions.

## Controller command line options

The controller itself accepts a few command line options. They allow debugging information to be shown and some options to be defined globally.

| Option | Description |
|---|---|
| `--disablePvInit` | Globally disable PV initialization. When disabled, the controller will not attempt to mount the NFS share to adjust directories and permissions before delivering the PV. |
| `--namespace`     | Restrict all StorageClasses to one particular namespace. If this value is defined, The `namespace` parameter in all StorageClasses will be ignored. |
| `--interval`      | Polling interval, in seconds, on the Kubernetes API. Default 30. |
| `--debugLevel`    | Adjust log level displayed on stdout. Possible values: error, warning, info, debug. Default: info. |
