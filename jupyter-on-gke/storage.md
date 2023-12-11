# Persistent Storage

## GCSFuse

**Important Note:** To use option, a GCS bucket must already be created within the project with the name in the format of `gcsfuse-{username}`

GCSFuse allow users to mount GCS Buckets as their local filesystem. This option allows ease of access on Cloud UI:

![Profiles Page](images/gcs_bucket.png)

Since this bucket in GCS, there is built in permission control and access outside of the clutser.

## Filestore

Filestore has built in auto provisioning. This means that the first time this option is picked for a users, it will auto provision a Filestore instance then mount it to the Jupyter instance.

**Important Note:** Currently the teir used is the `standard` tier. Different other teirs may require different handling.

Filestore can be accessed by any GCE instance within the same VPC network.

### Remounting a Filestore instance

A new Filestore instance will be created everytime the option is selected unless there is already an existing PVC under the format `cliam-{USERNAME}`. This means that to reuse an existing Filestore instance will require manual remounting.

1. Open up the Filestore page in [cloud console](https://console.cloud.google.com/filestore/instances). It will look something like this:

![Filestore instance](images/filestore_instance_screenshot.png)

2. Fill out both `filestire-pv.yaml` and `filestore-pvc.yaml` under the `persistent_storage_deployments` directory. The cloud console will have the necessary information.

3. Deploy the PVC/PV into the correct namespace (PV is cluster resource so this only applies to PVC)

4. Login to the user and select Filestore as persistent storage.