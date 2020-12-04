# minio-nfs
Minio container service using NFS persistent storage


### Create a new project for NFS
oc new-project minio-nfs

### Setup Service Account
oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:minio-nfs:minio-nfs

### Create Persistent volume
oc -n minio-nfs create -f minio-nfs-pv.yaml
oc -n minio-nfs create -f minio-nfs-pvc.yaml

### Create secret


### Create deployment
oc -n minio-nfs create -f minio-deployment.yaml

### Create deployment
oc -n minio-nfs create -f minio-service.yaml

