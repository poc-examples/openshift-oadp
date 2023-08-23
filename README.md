# openshift-oadp

### References

- [Backing up application Data](https://docs.openshift.com/container-platform/4.12/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/backing-up-applications.html)
### CRDs
- **VolumeSnapshotRestore**: Restores persistent volume data from a volume snapshot.
- **VolumeSnapshotBackup**: Backs up the data of a persistent volume as a volume snapshot.
- **DataProtectionApplication**: Represents applications and their data protection needs.
- **CloudStorage**: Specifies cloud storage configurations for backup storage.
- **VolumeSnapshotLocation**: Defines where volume snapshots are stored.
- **ServerStatusRequest**: Requests the status of the OADP operator's server.
- **Schedule**: Defines when and how frequently backups are taken.
- **Restore**: Initiates the restoration of data from a backup.
- **PodVolumeRestore**: Handles the restoration of pod volume data.
- **PodVolumeBackup**: Manages the backup of pod volume data.
- **DownloadRequest**: Requests download access to backup logs or data.
- **DeleteBackupRequest**: Requests the deletion of a specific backup.
- **BackupStorageLocation**: Specifies where backups are stored.
- **Backup**: Represents and manages a single backup instance.
- **BackupRepository**: Defines a storage location for storing backup artifacts.

### Installing the Operator

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: redhat-oadp-operator
  namespace: openshift-adp
spec:
  channel: stable-1.2
  installPlanApproval: Automatic
  name: redhat-oadp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: oadp-operator.v1.2.1
```
### Validating Installation

- Verify that the OADP operator pods are running

```
oc get pods -n openshift-adp
```

- Ensure that there are no errors in the operator logs

```
oc logs deploy/openshift-adp-controller-manager -n openshift-adp
```
### Configure Storage Locations

Create the Azure Blob Storage Account in advance.  Once the account is created OADP will create the blob container for you if it doesn't exist.  We will need to configure it to do this with a service principle.


### Create Service Principal

```
export GUID=<GUID>
export CLIENT_ID=<CLIENT_ID> 
export PASSWORD=<PASSWORD>  
export TENANT=<TENANT_ID> 
export SUBSCRIPTION_ID=<SUBSCRIPTION_ID> 
export RESOURCE_GROUP=<RESOURCE_GROUP_ID> 
```

```
az login --service-principal -u $CLIENT_ID -p $PASSWORD --tenant $TENANT
```

```
az ad sp create-for-rbac \
	--name "Velero" \
	--role "Contributor" \
	--scopes /subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}
```

### Create Secrets File

Create secrets file locally with the following information called credentials

```
AZURE_STORAGE_ACCOUNT_ACCESS_KEY: <base64-encoded-storage-account-key>
AZURE_SUBSCRIPTION_ID: <base64-encoded-subscription-id>
AZURE_TENANT_ID: <base64-encoded-tenant-id>
AZURE_CLIENT_ID: <base64-encoded-client-id>
AZURE_CLIENT_SECRET: <base64-encoded-client-secret>
AZURE_RESOURCE_GROUP: <base64-encoded-resource-group>
AZURE_CLOUD_NAME: <Base64EncodedString - "AzurePublicCloud">
```
### Create Storage Access Secrets

Create a Kubernetes secret in the `openshift-adp` namespace containing your Azure Storage Account Key in order to access Azure Blob Storage

```
oc create secret generic cloud-credentials-azure -n openshift-adp \
	--from-file cloud=credentials
```

### Create DataProtectionApplication

Once the operator is installed, you can create a `DataProtectionApplication` CR (Custom Resource). This CR is used to install Velero and its components on the cluster. When you create the DPA, it will trigger the installation of Velero, the Velero plugins for the storage provider (e.g., AWS, Azure), and other necessary components.

```
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: velero-dpa
  namespace: openshift-adp
spec:
  configuration:
    velero:
      featureFlags:
        - EnableCSI
      defaultPlugins:
        - azure
        - openshift
        - csi
      resourceTimeout: 10m
      ttl: 72h
    restic:
      enable: true
  backupLocations:
    - velero:
        config:
          resourceGroup: <resourceGroup>
          storageAccount: <storageAccount>
          subscriptionId: <subscriptionId>
          storageAccountKeyEnvVar: AZURE_STORAGE_ACCOUNT_ACCESS_KEY
        credential:
          key: cloud
          name: cloud-credentials-azure  
        provider: azure
        default: true
        objectStorage:
          bucket: <blobContainer> # Blob Container
          prefix: test
  snapshotLocations:
    - velero:
        config:
          resourceGroup: <resourceGroup>
          subscriptionId: <subscriptionId>
          incremental: "true"
          apiTimeout: 5m
        name: default
        provider: azure
```

### Validate Velero Installation

```
oc get pods -n openshift-adp
oc logs deploy/velero -n openshift-adp
```
### Label The Snapshot Class

```
 oc get volumesnapshotclass -o custom-columns=NAME:.metadata.name,PROVISIONER:.driver,LABELS:.metadata.labels
 oc label volumesnapshotclass csi-azuredisk-vsc velero.io/csi-volumesnapshot-class=true
```
### Test Installation with Manual Backup

```
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: manual-test-backup
  namespace: openshift-adp
  labels:
    velero.io/storage-location: <backupstoragelocation - velero-dpa-1>
spec:
  csiSnapshotTimeout: 10m0s
  defaultVolumesToFsBackup: false
  includedNamespaces:
    - <select-test-namespace>
  itemOperationTimeout: 1h0m0s
  storageLocation: <backupstoragelocation - velero-dpa-1>
  ttl: 720h0m0s
  volumeSnapshotLocations:
    - <backupstoragelocation - velero-dpa-1>
```

### Schedule Ongoing Backups

```
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: <cluster-name>-scheduled-backup
  namespace: openshift-adp
spec:
  schedule: '* */8 * * *'
  template:
    storageLocation: <backupstoragelocation - velero-dpa-1>
    includedNamespaces:
      - <list-of-namespaces|remove-for-all>
    excludedNamespaces:
      - openshift
```

### Attempt Restoration

```
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-<name-from-list-of-backups>
  namespace: openshift-adp
spec:
  backupName: <name-from-list-of-backups>
  excludedResources:
    - nodes
    - events
    - events.events.k8s.io
    - backups.velero.io
    - restores.velero.io
    - resticrepositories.velero.io
    - csinodes.storage.k8s.io
    - volumeattachments.storage.k8s.io
    - backuprepositories.velero.io
  itemOperationTimeout: 1h0m0s
  scheduleName: every-ten-minutes
```