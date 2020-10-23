# Backup and restore guide

## Create a manual backup

Backups are made on schedule, but if you want you can trigger one on-demand. Here's how:

```shell
### Trigger backup
EPOCH_TIME=$(date +%s)
kubectl create job upp-resources-autosaver-${EPOCH_TIME}-manual --from=cronjob/upp-resources-autosaver
### Wait for the backup to complete and look for the backup timestamp in the job pod
JOB_POD=$(kubectl get pod | grep upp-resources-autosaver-${EPOCH_TIME}-manual | awk '{print $1}')
kubectl logs ${JOB_POD} upp-resources-autosaver-pusher | tail
```

## Restore from backup

You need to be inside a [upp-eks-provisioner][upp-eks-provisioner] container to perform the exact
commands below.

```sh
### Ensure you are connected to the target cluster
eks-cluster-update-kubeconfig

### Find the appropriate autosave bucket and fill it in.
### It should be upp-resources-autosaver-$CLUSTER_UID
export AUTOSAVE_BUCKET="upp-resources-autosaver-${CLUSTER_UID}"
### Fill in the date from which you want to restore
export AUTOSAVE_BACKUP_DATE=""

# List for the backup you want to restore
aws s3 ls s3://${AUTOSAVE_BUCKET}/backup/ | grep "$AUTOSAVE_BACKUP_DATE"

### Fill in the backup directory name (e.g "2020-09-24_06-01-02/")
export BACKUP_TO_RESTORE=""
### Fill in the namespaces you want to restore
### WARNING: Do not restore the kube-system namespace
export NAMESPACES_TO_RESTORE="default prometheus"

upp-resources-autosaver-restore.sh \
  s3://${AUTOSAVE_BUCKET}/backup/${BACKUP_TO_RESTORE} \
  "$NAMESPACES_TO_RESTORE" 2>&1 | tee -a restore.log

# Review the restore.log
less restore.log
```

After this restore the `upp-prometheus` deployment will be most certainly
broken. Look for the "Redeploy upp-prometheus and the Prometheus service
monitors" guide in the [upp-prometheus docs][upp-prometheus-docs]

[upp-eks-provisioner]: https://github.com/Financial-Times/upp-eks-provisioner
[upp-prometheus-docs]: https://github.com/Financial-Times/upp-prometheus/blob/master/docs/troubleshooting.md
