# backup-velero

Chances that you lose some valuable information of your Kubernetes cluster *unintentionally* is somehow likely to happen at some point.

## Disaster Case Scenarios

1. Delete a namespace by accident.
2. Shutdown a Deployment without meaning to.
3. Specifying the wrong set of labels to a `kubectl delete` command, removing more than you intended.
4. Latest code changes introduced a critical bug that wiped a persistent volume and you lost the data.

## Introducing Velero

[Velero](https://velero.io) is an open source tool to safely backup and restore, perform disaster recovery, and migrate Kubernetes cluster resources and persistent volumes.

Velero runs in your cluster and connects to a storage service in the cloud (for instance, Amazon S3, or Azure Storage).

Features:

- **Back up Clusters**: Back up your Kubernetes resources and volumes for an entire cluster, or part of a cluster by using namespaces or label selectors.

- **Schedule Backups**: Set schedules to automatically kickoff backups at recurring intervals.

- **Backup Hooks**: Configure pre and post-backup hooks to perform custom operations before and after Velero backups.

## Installing and configuring Velero

Download Velero

```
$ wget https://github.com/vmware-tanzu/velero/releases/download/v1.2.0/velero-v1.2.0-linux-amd64.tar.gz

$ tar zxf velero-v1.2.0-linux-amd64.tar.gz

$ sudo mv velero-v1.2.0-linux-amd64/velero /usr/local/bin/

$ rm -rf velero*
```

## Set up server

These instructions start the Velero server and a Minio instance that is accessible from within the cluster at port 9000 and outside the cluster with a NodePort service at port 30001.

Minio is a local S3-compatible object storage service. It provides a convenient way to test Velero without tying you to a specific cloud provider.

Create a Velero-specific credentials file (credentials-velero) in your local directory:

```
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
```

Start the server and the local storage service. In the Velero directory, run:

```
$ kubectl apply -f ./src/00-minio-deployment.yaml

$ velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.0.0 \
    --bucket velero \
    --secret-file ./src/credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
```

You can access the Minio dashboard at http://127.0.0.1:30001

Deploy the example nginx application:

```
$ kubectl apply -f src/nginx-app/base.yaml
```
Check to see that both the Velero and nginx deployments are successfully created:

```
$ kubectl get deployments -l component=velero --namespace=velero

$ kubectl get deployments --namespace=nginx-example
```

## Back up

Create a backup for any object that matches the app=nginx label selector:

```
$ velero backup create nginx-backup --selector app=nginx
```

Alternatively if you want to backup all objects except those matching the label backup=ignore:

```
$ velero backup create nginx-backup --selector 'backup notin (ignore)'
```

(Optional) Create regularly scheduled backups based on a cron expression using the app=nginx label selector:

```
$ velero schedule create nginx-daily --schedule="0 1 * * *" --selector app=nginx
```

Alternatively, you can use some non-standard shorthand cron expressions:

```
$ velero schedule create nginx-daily --schedule="@daily" --selector app=nginx
```

See the cron package's documentation for more usage examples.

 💣💥 Simulate a disaster:

```
$ kubectl delete namespace nginx-example
```

To check that the nginx deployment and service are gone, run:

```
$ kubectl get deployments --namespace=nginx-example

$ kubectl get services --namespace=nginx-example

$ kubectl get namespace/nginx-example
```
You should get no results.

> NOTE: You might need to wait for a few minutes for the namespace to be fully cleaned up.

## Restore
Run:

```
$ velero restore create --from-backup nginx-backup
```

Run:

```
$ velero restore get
```

After the restore finishes, the output looks like the following:

```
NAME                          BACKUP         STATUS      WARNINGS   ERRORS    CREATED                         SELECTOR
nginx-backup-20170727200524   nginx-backup   Completed   0          0         2020-01-03 20:05:24 +0000 UTC   <none>
```

> NOTE: The restore can take a few moments to finish. During this time, the STATUS column reads InProgress.

After a successful restore, the STATUS column is Completed, and WARNINGS and ERRORS are 0. All objects in the nginx-example namespace should be just as they were before you deleted them.

If there are errors or warnings, you can look at them in detail:

```
$ velero restore describe <RESTORE_NAME>
```

## Clean up

If you want to delete any backups you created, including data in object storage and persistent volume snapshots, you can run:

```
$ velero backup delete BACKUP_NAME
```

This asks the Velero server to delete all backup data associated with BACKUP_NAME. You need to do this for each backup you want to permanently delete. A future version of Velero will allow you to delete multiple backups by name or label selector.

Once fully removed, the backup is no longer visible when you run:

```
$ velero backup get BACKUP_NAME
```

To completely uninstall Velero, minio, and the nginx example app from your Kubernetes cluster:

```
$ kubectl delete namespace/velero clusterrolebinding/velero

$ kubectl delete crds -l component=velero

$ kubectl delete -f src/nginx-app/base.yaml
```

## Links
- [Velero v.1.2.0 docs](https://velero.io/docs/v1.2.0/contributions/minio/)