# OADP + Minio Demo

A demo repository to show how OADP works with a generic S3 backend (Minio).
The purpose of this demo is purely to showcase the features of OADP using a simple
backend during labs and workshops.

[OADP](https://github.com/konveyor/oadp-operator) (OpenShift APIs for Data Protection) 
is an operator that Red Hat has created to create backup and restore APIs in the 
OpenShift cluster. OADP is based on Velero.

[MinIO](https://github.com/minio/minio) is a High Performance Object Storage which 
is API compatible with Amazon S3 cloud storage service.


## Install OADP from the OpenShift Console
Install the OADP Operator from the Openshift's OperatorHub. Search for the operator using keywords like `oadp` or `velero`

![OADP-OLM-1](docs/images/OADP-OLM-1.png)

Now click on Install

![OADP-OLM-2](docs/images/OADP-OLM-2.png)

Finally, click on subscribe, this will create a namespace named `oadp-operator` if it does not exist and install the OADP operator in it.

![OADP-OLM-3](docs/images/OADP-OLM-3.png)

## Install OADP from the CLI (Console alternative)
Create the `oadp-operator` namespace.
```
oc create ns oadp-operator
```

Create the OperatorGroup to let the OLM manage the operator in the namespace:
```
oc apply -f resources/oadp-operatorgroup.yaml
```

Create the operator subscription:
```
oc apply -f resources/oadp-subscription.yaml
```

Wait for the installation to complete.

## Install Minio

Create the `minio` namespace:
```
oc create ns minio
```

Install Minio using Helm in the `minio` namespace.
```
helm install -n minio minio minio/minio
```

Expose the minio service as a secure route.
```
oc create route edge minio --service=minio
```

Obtain the Access and Secret from the installed chart. 
```
export ACCESS_KEY=$(kubectl get secret minio -o jsonpath="{.data.accesskey}" | base64 --decode)
export SECRET_KEY=$(kubectl get secret minio -o jsonpath="{.data.secretkey}" | base64 --decode)
```

Generate the `credentials` file using the exported variables:
```
cat > credentials << EOF
[default]
aws_access_key_id=$ACCESS_KEY
aws_secret_access_key=$SECRET_KEY
EOF
```

Generate the `cloud-credentials` secret in the `oadp-operator` namespace to let Velero authenticate
with the S3 backend.
```
oc create secret generic cloud-credentials --namespace oadp-operator --from-file cloud=credentials
```

After installing Minio, and **before** installing the Velero CR, create a 
target bucket inside minio. The bucket name should match with the one defined
in the Velero CR. This demo uses a bucket called `example`.

## Create the Velero resource

The Velero CR is customized to use the generated route. Customize the `s3_url` field with the exposed route.
```
apiVersion: konveyor.openshift.io/v1alpha1
kind: Velero
metadata:
  name: example-velero
  namespace: oadp-operator
spec:
  olm_managed: true
  backup_storage_locations:
    - config:
        profile: default
        region: minio
        s3_url: https://minio-minio.apps.ocp4.rhocplab.com
        s3_force_path_style: true
        insecure_skip_tls_verify: true
      credentials_secret_ref:
        name: cloud-credentials
        namespace: oadp-operator
      name: default
      object_storage:
        bucket: example
        prefix: velero
      provider: aws
  default_velero_plugins:
    - aws
    - csi
    - openshift
  velero_feature_flags: EnableCSI
  enable_restic: true
  volume_snapshot_locations:
    - config:
        profile: default
        region: minio
      name: default
      provider: aws
```

Create the Velero CR.
```
oc apply -f resources/example-velero.yaml
```

## Create a sample backup namespace
```
oc new-project example-namespace
oc new-app --template=nodejs-mongo-persistent
```

Wait for the app build completion.
```
$ oc get pods
NAME                               READY   STATUS      RESTARTS   AGE
mongodb-1-deploy                   0/1     Completed   0          2m35s
mongodb-1-lkr8v                    1/1     Running     0          2m31s
nodejs-mongo-persistent-1-6l24f    1/1     Running     0          55s
nodejs-mongo-persistent-1-build    0/1     Completed   0          2m35s
nodejs-mongo-persistent-1-deploy   0/1     Completed   0          60s
```

Verify a PVC was successfully bounded:
```
$ oc get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
mongodb   Bound    pvc-4eae7794-4117-4d37-969d-58702e757eac   1Gi        RWO            managed-nfs-storage   3m27s
```

## Create the backup resource
The backup resource defines the backup parameteres, the storage location (which
is the S3 backend configured in the Velero CR) and the included namespaces.
It is also possible to configure exclusion lists.
The `snapshotVolumes` boolean can be used to enable/disable the volume snapshots (this will
imply a CSI storage driver).
```
apiVersion: velero.io/v1
kind: Backup
metadata:
  namespace: oadp-operator
  name: example-backup
spec:
  storageLocation: default
  snapshotVolumes: true
  includedNamespaces:
    - example-namespace
```

Apply the backup resource.
```
oc apply -f resources/example-backup.yaml
```

Verify the backup was completed successuflly by looking at the objects created
in the Minio console:

![Minio-Backup](docs/images/minio-example-backup.png)

**NOTE**: If the storage backend does not support CSI snapshots the volume snapshop will be skipped.
The following kind of message will appear in the Velero logs:
```
time="2021-06-15T18:24:29Z" level=info msg="Persistent volume is not a supported volume type for snapshots, skipping." backup=oadp-operator/example-backup logSource="pkg/backup/item_backupper.go:469" name=pvc-a101734c-f9eb-4f18-9936-57453a88b69c namespace= persistentVolume=pvc-a101734c-f9eb-4f18-9936-57453a88b69c resource=persistentvolumes
```

## Links
- https://access.redhat.com/articles/5456281
- https://www.openshift.com/blog/hybrid-cloud-disaster-recovery-on-openshift

## Maintainers
Gianni Salinetti <gsalinet@redhat.com>  

