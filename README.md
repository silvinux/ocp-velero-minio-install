# ocp-velero-minio-install
## Create NFS shares for Minio Persistent Storage. In my case, I used the bastion server.

$ sudo mkdir /var/nfsshare/velero/minio-{config,storage}/
$ sudo chown nfsnobody:nfsnobody /var/nfsshare/velero/minio-{config,storage}/
$ sudo chmod 777 /var/nfsshare/velero/minio-{config,storage}/
$ cat /etc/exports.d/velero-minio.exports
"/var/nfsshare/velero/minio-config" *(rw,no_root_squash)
"/var/nfsshare/velero/minio-storage" *(rw,no_root_squash)
$ sudo exportfs -rva

## Clone the repository
$ git clone https://github.com/silvinux/ocp-velero-minio-install.git

## Deploy the Minio server from the template. Change the variables accordingly.
$ oc process -f minio-deployment-template.yaml USER_MINIO_ACCESS_KEY=minio PASSWORD_MINIO_SECRET_KEY=minio123 \ 
MINIO_ACCESS_ROUTE=minio.apps.lab.example.com NFS_SEVER=nfs-lb.lab.example.com \
PATH_MINIO_CONFIG_NFS_EXPORT=/var/nfsshare/minio/config SIZE_MINIO_CONFIG_NFS_EXPORT=1Gi \
PATH_MINIO_STORAGE_NFS_EXPORT=/var/nfsshare/minio/storage SIZE_MINIO_STORAGE_NFS_EXPORT=3Gi |oc create -f 


## Create the file credentials-velero, which contains the minio's user/passwd.
$ cat credentials-velero
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123

## Download the Velero binary
$ wget https://github.com/vmware-tanzu/velero/releases/download/v1.2.0/velero-v1.2.0-linux-amd64.tar.gz
$ tar xvfz velero-v1.2.0-linux-amd64.tar.gz
$ cp velero-v1.2.0-linux-amd64/velero /usr/local/bin/

## Install valero, pointing to the bucket created in Minio server. We're going to use Velero's restic integration.
$ velero install \
--provider aws --bucket velero \
--secret-file ./credentials-velero \
--use-volume-snapshots=false \
--backup-location-config region=minio,s3ForcePathStyle="true",publicUrl=http://minio.apps.lab.example.com,s3Url=http://minio.velero.svc:9000 \
--plugins velero/velero-plugin-for-aws:v1.0.0 \
--use-volume-snapshots=false \
--use-restic

## The restic containers should be running in a privileged mode to be able to mount the correct hostpath to pods volumes.

$ oc adm policy add-scc-to-user privileged -z velero -n velero

$ oc patch ds/restic \
  --namespace velero \
  --type json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/securityContext","value": { "privileged": true}}]'

$ oc patch ds/restic \
  --namespace velero \
  --type json \
  -p '[{"op":"replace","path":"/spec/template/spec/volumes/0/hostPath","value": { "path": "/var/lib/origin/openshift.local.volumes/pods"}}]'

## Wait until the pods are running.
$ oc get pods -o wide
NAME                      READY     STATUS      RESTARTS   AGE       IP            NODE      NOMINATED NODE
minio-56554f45b9-7bpxp    1/1       Running     4          26m       10.130.0.45   node1     <none>
minio-setup-6hwmv         0/1       Completed   4          26m       10.130.0.44   node1     <none>
restic-8gdrj              1/1       Running     0          20m       10.130.0.49   node1     <none>
restic-pt9jp              1/1       Running     0          20m       10.129.0.77   node2     <none>
velero-77b4587448-9fvth   1/1       Running     0          21m       10.130.0.46   node1     <none>

## We MUST add an annotation to the pods we want to make a backup of the PVC, this way Velero will be aware of which PVCs he must to backup in a specific namespace. Instead doing in the pod itself, because it's ephemeral, I rather to do it in the DeploymentConfig.

$ ./get_pods_mounpoint.sh gogs
-------------------------------
POD: gogs-4-6lvr5
MountPoint: /data
VolumeName: gogs-volume-1
oc rsync gogs-4-6lvr5:/data
-------------------------------
-------------------------------
POD: postgresql-2-xbsnl
MountPoint: /var/lib/pgsql/data
VolumeName: postgresql-data
oc rsync postgresql-2-xbsnl:/var/lib/pgsql/data

### Annotate the pod. 
$ oc annotate pod -n gogs-backup --selector=deploymentconfig=gogs backup.velero.io/backup-volumes=gogs-volume-1,config-volume --overwrite
$ oc annotate pod -n gogs-backup --selector=deploymentconfig=postgresql backup.velero.io/backup-volumes=postgresql-data --overwrite

### Annotate the DeploymentConfig. 
$ oc patch dc/gogs -p '{"spec":{"template":{"metadata":{"annotations":{"backup.velero.io/backup-volumes": "gogs-volume-1"}}}}}'  -n gogs
$ oc patch dc/postgresql -p '{"spec":{"template":{"metadata":{"annotations":{"backup.velero.io/backup-volumes": "postgresql-data"}}}}}'  -n gogs


## Create a backup.
$ velero backup create gogs-backup --include-namespaces gogs

## Check backup's details.
$ velero backup describe gogs-backup --details

## Check backup's logs.
$ velero backup logs gogs-backup

## Schedule a backup. Velero use same format as a cronjob.
$ velero schedule create daily-gogs --schedule="*/5 * * * *" --include-namespaces gogs

## Restore a project. If dinamic provisioning is not active in cluster, you should provide a PV before start the restore.
$ velero restore create --from-backup gogs-backup

## To restore a backed up namespace onto a new namespace (Eg to run the two side by side).
$ velero restore create --from-backup full-backup-[namespace] --namespace-mappings [old-namespace]:[new-namespace]

#### Sources.
https://github.com/vmware-tanzu/
https://github.com/vmware-tanzu/velero-plugin-for-aws
https://velero.io/docs/master/
https://velero.io/docs/master/restic/
https://blog.openshift.com/backup-openshift-resources-the-native-way/
