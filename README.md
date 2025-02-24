# OADP Demo

An Intro to OpenShift API for Data Protection (OADP).

## Intro

This demo highlights the basics for backing up and restoring a user namespace.  It uses OpenShift Data Foundation for storage classes and Object Storage.  In a production-ready environment, you would want to have object storage that is **external** to your OpenShift cluster.

## Prerequisites

This demo assumes ODF has been deployed to your cluster.  In order for OADP to backup PVCs with CSI snapshots, you require a `VolumeSnapshotClass` with [the following labels and annotations](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/backup_and_restore/oadp-application-backup-and-restore#oadp-backing-up-pvs-csi-doc):

```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: <volume_snapshot_class_name>
  labels:
    velero.io/csi-volumesnapshot-class: 'true'
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: 'true'
driver: <csi_driver>
deletionPolicy: <deletion_policy_type> 
```

## Deploying and Configuring OADP

### Step 1:  Install OADP Operator

You can do this with the CLI or the OpenShift console.  [Follow the documentation to deploy the OADP operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/backup_and_restore/oadp-application-backup-and-restore#installing-and-configuring-oadp).

### Step 2:  Create an ObjectBucket

OADP requires object storage to store backups.  This would normally be a storage service that is external to your cluster, for example:

* AWS S3
* Azure Blob
* On-prem storage that provides S3 interface
* Another OCP cluster that exposes an S3 interface (MCG/Minio/etc...)

For this demo, we will simply use an ObjectBucket in the same cluster.  Create this bucket with the following command:

```
oc apply -f gitops/objectbucket
```

### Step 3:  ObjectBucket Credentials Secret

With the bucket created, you need to create a `Secret` with the key id / access key values:

```
echo "aws_access_key_id: "
echo $(oc get secret backup-bucket -n openshift-adp -o go-template --template="{{.data.AWS_ACCESS_KEY_ID|base64decode}}")

echo "aws_secret_access_key:"
echo $(oc get secret backup-bucket -n openshift-adp -o go-template --template="{{.data.AWS_SECRET_ACCESS_KEY|base64decode}}")
```

Based on the output of the above command, complete and create the following `Secret`:

```
apiVersion: v1
kind: Secret
metadata:
  name: cloud-credentials
  namespace: openshift-adp
type: Opaque
stringData:
  cloud: |
    [default]
    aws_access_key_id=<your aws access key id>
    aws_secret_access_key=<your aws secret access key>
```

### Step 4: DataProtectionApplication

Finally, you need to create a `DataProtectionApplication`.  This will configure OADP and enable backup/restore functionality.

First, you'll need the name of the object bucket you created.  You can find this by running the following command:

```
echo $(oc get cm backup-bucket -n openshift-adp -o go-template --template="{{.data.BUCKET_NAME}}")
```

Add this to `gitops/dpa/dpa.yaml` before applying the CRD:

```
oc apply -f gitops/dpa
```

After a minute or two, the DataProtectionApplication should be ready.  You can confirm this by checking its status with the following command:

```
oc get dpa default-dpa -n openshift-adp -o yaml
```

This will also create a new `BackupStorageLocation`.  You can check to see if it is in the **Available** phase with the command:

```
oc get bsl default-dpa-1 -n openshift-adp -o yaml
```

## Deploy A Demo Application

You're going to need something to backup!  Run the following command to deploy an app that has a `PersistentVolumeClaim`:

```
oc apply -k gitops/deployment-oadp-demo
```

And lets add some data to that PV:

```
POD_NAME=$(oc get pods -o name -n deployment-oadp-demo | cut -d'/' -f2-)

oc cp -n deployment-oadp-demo  gitops/deployment-oadp-demo/config.properties deployment-oadp-demo/$POD_NAME:/data/config/config.properties
```

Now there's some data in the PVC!

You can see this either through the CLI or in the OpenShift UI by navigating to the `deployment-oadp-demo` namespace and opening a terminal on the pod, then navigating to the `/data/config` dir and running `ls -la` or `cat config.properties`.

## Backup the Application

Now that there is an app with data, let's back it up.

This is done with a `Backup` CRD.  You can read more about the [options available here](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/backup_and_restore/oadp-application-backup-and-restore#oadp-creating-backup-cr-doc).

This "one shot" backup CRD is pretty simple:

```
kind: Backup
apiVersion: velero.io/v1
metadata:
  name: backup1
  namespace: openshift-adp
spec:
  csiSnapshotTimeout: 10m0s
  defaultVolumesToFsBackup: false
  includedNamespaces:
    - deployment-oadp-demo
  itemOperationTimeout: 4h0m0s
  snapshotMoveData: true 
  storageLocation: default-dpa-1
  ttl: 720h0m0s 
```

## Delete The Application And Restore It

Time to test things out!

First, delete the application:

```
oc delete project deployment-oadp-demo
```

Once that's done, you can create the `Restore` CRD to restore the namespace, app, and PVC:

```
oc create -f gitops/restore
```

If your'e fast enough, you can watch the progress of the restore in the OpenShift console.

Once the restore is done, you can verify that the application is running again, and that the data that you uploaded is still in the PVC.

## Schedules

Most of the time, you would use a `Schedule` instead so that you can run backups at a specific time/cadence:

```
apiVersion: velero.io/v1
kind: Schedule
metadata:
  namespace: openshift-adp
  name: deployment-oadp-demo
  labels:
    app.kubernetes.io/instance: deployment-oadp-demo
spec:
  schedule: '*/20 * * * *'
  template:
    hooks: {}
    csiSnapshotTimeout: 10m0s
    defaultVolumesToFsBackup: false
    includedNamespaces:
      - deployment-oadp-demo
    itemOperationTimeout: 4h0m0s
    snapshotMoveData: true 
    storageLocation: default-dpa-1
    ttl: 720h0m0s
  useOwnerReferencesInBackup: false
```

