# 1- NFS server is set up 
Longhorn Volume DR: Cluster A ↔ Cluster B

# 2- Prepare the two RKE2 clusters
Cluster A = primary/production.
Cluster B = DR/secondary.
Install the same Longhorn version on both clusters. 
Make sure both clusters have:
Longhorn installed and healthy.
At least one usable Longhorn disk.
CSI functioning.
Network connectivity to the backup target.
Use the same backup target on both clusters, for example:
nfs://192.168.122.86:/home/nfsshare

Configure the backup target in Longhorn on both clusters.
Confirm the target is healthy/available before proceeding.
Longhorn recommends using a reliable external backup target; NFS is supported.

# 3- Create the application on Cluster A
For example, deploy:
```
dr-test
 ├── Deployment
 ├── Service
 └── PVC
       │
       ▼
   Longhorn Volume
```
Example:
```
apiVersion: v1
kind: Namespace
metadata:
  name: dr-test
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-data
  namespace: dr-test
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dr-test
  namespace: dr-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dr-test
  template:
    metadata:
      labels:
        app: dr-test
    spec:
      containers:
      - name: app
        image: nginx:alpine
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: test-data
```
Apply it on Cluster A:
kubectl apply -f app.yaml

Verify:
kubectl get pods,pvc -n dr-test

Verify the Longhorn volume (Or from UI):
kubectl get volumes.longhorn.io -n longhorn-system

# 4- Put identifiable data into the application
Do not just rely on the application's existing data.

Create a clearly identifiable marker:

kubectl exec -n dr-test deploy/dr-test -- \
  sh -c 'date -u > /usr/share/nginx/html/data/test.txt'

Or:

kubectl exec -n dr-test deploy/dr-test -- \
  sh -c 'echo "DATA CREATED ON CLUSTER A" > /usr/share/nginx/html/data/test.txt'

Verify:

kubectl exec -n dr-test deploy/dr-test -- \
  cat /usr/share/nginx/html/data/test.txt

You should see:

DATA CREATED ON CLUSTER A

# 5- Configure recurring Longhorn backup job on Cluster A
Create a Longhorn recurring backup job.

For example:
```
Name: dr-backup
Task: backup
Schedule: */15 * * * *
Retain: 96
```
You can configure this through the Longhorn UI or Kubernetes.

Then associate the job with the application volume (important) == go to volume, then scroll down to jobs and create recurring job. 

For example, verify:

kubectl get recurringjob.longhorn.io \
  -n longhorn-system

And:

kubectl get volume.longhorn.io \
  -n longhorn-system

The volume should have the recurring backup association.

# 6- Verify the first backup on Cluster A use UI or : 
Don't proceed until you have a successful backup.

Run:

kubectl get backupvolume.longhorn.io \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,LASTBACKUP:.status.lastBackupName,LASTBACKUPAT:.status.lastBackupAt,LASTSYNC:.status.lastSyncedAt'

Then:

kubectl get backups.longhorn.io \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,CREATED:.status.backupCreatedAt,STATE:.status.state,URL:.status.url'

You want:

STATE
Completed

and a valid backup URL such as:

nfs://192.168.122.86:/home/nfsshare?backup=...&volume=...

# 7- Configure the same application on Cluster B only deployment without PV and PVC

The desired architecture is:
```
                 NFS Backup Target
                /                 \
               /                   \
      Cluster A                    Cluster B
      ---------                    ---------
      Application                  Application
          |                            |
      PVC / Volume                 DR Volume
          |                            |
      Longhorn                     Longhorn
          \                            /
           \------ backups -----------/
```
# 8- Create the DR volume on Cluster B
On Cluster B, create a Longhorn volume from the latest backup. from UI

Conceptually:
```
apiVersion: longhorn.io/v1beta2
kind: Volume
metadata:
  name: dr-test-data
  namespace: longhorn-system
spec:
  size: "5368709120"
  numberOfReplicas: 1
  accessMode: rwo
  frontend: blockdev
  fromBackup: "nfs://192.168.122.86:/home/nfsshare?backup=<LATEST-BACKUP>&volume=<SOURCE-VOLUME>"
  Standby: true
```
Use the exact backup URL from the backup object.
Don't manually guess the backup ID.
Check:
kubectl get backups.longhorn.io -n longhorn-system

After creation, monitor:
kubectl get volume.longhorn.io dr-test-data \
  -n longhorn-system -w

Initially, you may see:
RestoreInProgress

Wait until:
restoreRequired=false and the volume is healthy.

The backup restore mechanism reconstructs the Longhorn volume from the backup data. Longhorn backups are incremental and use change-block tracking. 

# 9- Verify the DR volume before failover
Check:

kubectl get volume.longhorn.io dr-test-data \
  -n longhorn-system \
  -o jsonpath='{.status.state}{" | "}{.status.robustness}{" | "}{.status.restoreRequired}{"\n"}'

Ideally:

detached | healthy | false

# 10 - Perform Cluster A → Cluster B failover
Before failover:

Stop writes on Cluster A.
Ideally scale the application down:
kubectl scale deployment dr-test \
  -n dr-test \
  --replicas=0

Wait for the final backup to complete.
Record the final backup ID:
kubectl get backupvolume.longhorn.io \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,LASTBACKUP:.status.lastBackupName,LASTBACKUPAT:.status.lastBackupAt'

For a real disaster, you may not be able to perform the clean shutdown. That's where your RPO comes into play.

# 11- On Cluster B:

Restore/create the DR volume from the latest available backup.
create reccuring backup job from the volume => go to volume => scroll down => use exiting job /or create
Activate Volume 
Attach/mount the volume through the B-side PV/PVC.
Scale the application up:
kubectl scale deployment dr-test \
  -n dr-test \
  --replicas=1

Verify:
kubectl get pods -n dr-test

Verify data:
kubectl exec -n dr-test <pod> -- \
  cat /usr/share/nginx/html/data/test.txt

At this point:
```
          FAILOVER
             ↓
Cluster A ─────────X
                    \
                     \
                      → Cluster B
                           |
                           +-- application
                           |
                           +-- restored Longhorn volume
```
# 12- Write new data on Cluster B
kubectl exec -n dr-test <pod> -- \
  sh -c 'echo "DATA CREATED ON CLUSTER B AFTER FAILOVER" >> /usr/share/nginx/html/data/test.txt'

Verify:

kubectl exec -n dr-test <pod> -- \
  cat /usr/share/nginx/html/data/test.txt

You should now have:

DATA CREATED ON CLUSTER A
DATA CREATED ON CLUSTER B AFTER FAILOVER

# 13- let cluster B take a new backup
This is the critical step for failback.

Cluster B must successfully back up the changed DR volume to the shared backup target.

Check:

kubectl get backupvolume.longhorn.io \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,LASTBACKUP:.status.lastBackupName,LASTBACKUPAT:.status.lastBackupAt,LASTSYNC:.status.lastSyncedAt'

Then:

kubectl get backups.longhorn.io \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,CREATED:.status.backupCreatedAt,STATE:.status.state,URL:.status.url'

You want a new backup:

backup-a35af0bee6084662 for example.

And: STATE=Completed

Do not start failback until this is confirmed.

# 13- Stop writes on Cluster B
Before failback:

kubectl scale deployment dr-test \
  -n dr-test \
  --replicas=0

Wait until:

kubectl get pods -n dr-test

shows no application pod writing to the volume.

Then confirm the final B backup completed.

# 14- On Cluster A, sync the backup target
Cluster A needs to discover the backup created by Cluster B.

Check:

kubectl get backupvolume.longhorn.io \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,LASTBACKUP:.status.lastBackupName,LASTBACKUPAT:.status.lastBackupAt,LASTSYNC:.status.lastSyncedAt'

The important point is:
```
Cluster B
    ↓
backup-a35...
    ↓
NFS
    ↓
Cluster A backup target
    ↓
backup-a35...
```
If Cluster A does not see the new backup, stop here.

Do not restore from the old backup.

Force/check backup target synchronization according to your Longhorn configuration and verify the new backup is visible.

# 15- Create a new failback volume on Cluster A -> create DR volume from new backup on cluster A + attach reccuring job to the volume
Create a new volume, for example:

dr-test-data-failback

from:

backup-a35af0bee6084662

Conceptually:

spec:
  fromBackup: "nfs://192.168.122.86:/home/nfsshare?backup=backup-a35af0bee6084662&volume=pvc-..."

Monitor:

kubectl get volume.longhorn.io dr-test-data-failback \
  -n longhorn-system -w

# 16- Point the Cluster A application at the failback volume - activate volume and create PVc/PV
Once the restored volume has been independently verified:
```
Cluster A
   |
   +-- Application
          |
          +-- PVC
                |
                +-- PV
                      |
                      +-- dr-test-data-failback
```
The important thing is that the application's PVC must reference the restored failback PV, not:

pvc-c1997f6f-...

from the old Cluster A volume.

# 17- Start the application on Cluster A and activate volume on cluster A 
Once the PVC is:

Bound

start the application:

kubectl scale deployment dr-test \
  -n dr-test \
  --replicas=1

Then:

kubectl get pods -n dr-test -w

Wait for:

Running

Then verify:

kubectl exec -n dr-test <pod> -- \
  cat /usr/share/nginx/html/data/test.txt

Expected:

DATA CREATED ON CLUSTER A
DATA CREATED ON CLUSTER B AFTER FAILOVER

That proves:
```
                 FAILOVER

Cluster A
    │
    │ backup
    ▼
   NFS
    │
    ▼
Cluster B
    │
    │ new data
    │ backup
    ▼
   NFS
    │
    ▼
Cluster A
    │
    ▼
failback volume
    │
    ▼
application
```
# 18- Final failback verification
After the application is running on Cluster A:

Read the data.
Write another unique marker:
kubectl exec -n dr-test <pod> -- \
  sh -c 'echo "BACK ON CLUSTER A" >> /usr/share/nginx/html/data/test.txt'

Verify:
kubectl exec -n dr-test <pod> -- \
  cat /usr/share/nginx/html/data/test.txt

You should have:

DATA CREATED ON CLUSTER A
DATA CREATED ON CLUSTER B AFTER FAILOVER
BACK ON CLUSTER A

NOTE: before taking down the app you should wait for at least 1 minute for the backup to detect the changes you made. 
