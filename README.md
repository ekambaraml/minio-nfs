## Minio Deployment in OpenShift

```yaml
cat <<EOF |oc apply -f -
---
apiVersion: v1
kind: Namespace
metadata:
  name: minio

---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: minio
  name: minio
  labels:
    component: minio
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      component: minio
  template:
    metadata:
      labels:
        component: minio
    spec:
      volumes:
      - name: storage
        emptyDir: {}
      - name: config
        emptyDir: {}
      containers:
      - name: minio
        image: minio/minio:latest
        imagePullPolicy: IfNotPresent
        args:
        - server
        - /storage
        - --config-dir=/config
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        ports:
        - containerPort: 9000
        volumeMounts:
        - name: storage
          mountPath: "/storage"
        - name: config
          mountPath: "/config"

---
apiVersion: v1
kind: Service
metadata:
  namespace: minio
  name: minio
  labels:
    component: minio
spec:
  # ClusterIP is recommended for production environments.
  # Change to NodePort if needed per documentation,
  # but only if you run Minio in a test/trial environment, for example with Minikube.
  type: ClusterIP
  ports:
    - port: 9000
      targetPort: 9000
      protocol: TCP
  selector:
    component: minio

---
apiVersion: batch/v1
kind: Job
metadata:
  namespace: minio
  name: minio-setup
  labels:
    component: minio
spec:
  template:
    metadata:
      name: minio-setup
    spec:
      restartPolicy: OnFailure
      volumes:
      - name: config
        emptyDir: {}
      containers:
      - name: mc
        image: minio/mc:latest
        imagePullPolicy: IfNotPresent
        command:
        - /bin/sh
        - -c
        - "mc --config-dir=/config config host add velero http://minio:9000 minio minio123 && mc --config-dir=/config mb -p velero/velero"
        volumeMounts:
        - name: config
          mountPath: "/config"
          
EOF



oc set image deployment/minio minio=minio/minio:RELEASE.2021-06-17T00-10-46Z -n minio

cat <<EOF |oc apply -f -

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: minio
  name: minio-config-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: managed-nfs-storage

EOF


cat <<EOF |oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: minio
  name: minio-storage-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
  storageClassName: managed-nfs-storage
EOF


oc set volume deployment.apps/minio --add --overwrite --name=config --mount-path=/config --type=persistentVolumeClaim --claim-name="minio-config-pvc" -n minio


oc set volume deployment.apps/minio --add --overwrite --name=storage --mount-path=/storage --type=persistentVolumeClaim --claim-name="minio-storage-pvc" -n minio

oc set resources deployment minio -n minio --requests=cpu=500m,memory=256Mi --limits=cpu=1,memory=1Gi

cat <<EOF |oc apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  namespace: minio
  name: minio-setup
  labels:
    component: minio
spec:
  template:
    metadata:
      name: minio-setup
    spec:
      restartPolicy: OnFailure
      volumes:
      - name: config
        emptyDir: {}
      containers:
      - name: mc
        image: minio/mc:latest
        imagePullPolicy: IfNotPresent
        command:
        - /bin/sh
        - -c
        - "mc --config-dir=/config config host add velero http://minio:9000 minio minio123 && mc --config-dir=/config mb -p velero/velero"
        volumeMounts:
        - name: config
          mountPath: "/config"
          
EOF



9.	Get the MinIO URL 

oc get route minio -n velero
 
Example: 
http://minio-minio.apps.aimlops.cp.fyre.ibm.com 

User/password 
minio/minio123

