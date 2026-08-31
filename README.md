NFS server is set up 
Longhorn Volume DR: Cluster A ↔ Cluster B
1. Prepare the two RKE2 clusters
Cluster A = primary/production.
Cluster B = DR/secondary.
Install the same Longhorn version on both clusters. For your environment, that means Longhorn v1.11.3.
Make sure both clusters have:
Longhorn installed and healthy.
At least one usable Longhorn disk.
CSI functioning.
Network connectivity to the backup target.
Use the same backup target on both clusters, for example:
nfs://192.168.122.86:/home/nfsshare

Configure the backup target in Longhorn on both clusters.
Verify it:
kubectl get backuptarget.longhorn.io -n longhorn-system

Verify:
kubectl get backuptarget.longhorn.io default \
  -n longhorn-system -o yaml

Confirm the target is healthy/available before proceeding.
Longhorn recommends using a reliable external backup target; NFS is supported, although object storage is generally preferable for production. 

2. Create the application on Cluster A
For example, deploy:

dr-test
 ├── Deployment
 ├── Service
 └── PVC
       │
       ▼
   Longhorn Volume

Example:

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

Apply it on Cluster A:
kubectl apply -f app.yaml

Verify:
kubectl get pods,pvc -n dr-test

Verify the Longhorn volume:
kubectl get volumes.longhorn.io -n longhorn-system

3. Put identifiable data into the application
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

4. Configure recurring Longhorn backups on Cluster A
Create a Longhorn recurring backup job.

For example:

Name: dr-backup
Task: backup
Schedule: */15 * * * *
Retain: 96

You can configure this through the Longhorn UI or Kubernetes.

Then associate the job with the application volume.

For example, verify:

kubectl get recurringjob.longhorn.io \
  -n longhorn-system

And:

kubectl get volume.longhorn.io \
  -n longhorn-system

The volume should have the recurring backup association.
5. Verify the first backup on Cluster A
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

6. Configure the same application on Cluster B
This is critical.

Longhorn does not deploy your application on Cluster B.

You need to deploy the Kubernetes application manifests separately.

However, don't create the normal application PVC against a blank Longhorn volume on B.

The DR volume should be created from the backup.

The desired architecture is:

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

7. Create the DR volume on Cluster B
On Cluster B, create a Longhorn volume from the latest backup. from UI

Conceptually:

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
  standby: true

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
restoreRequired=false

and the volume is healthy.

The backup restore mechanism reconstructs the Longhorn volume from the backup data. Longhorn backups are incremental and use change-block tracking. 
L
Longhorn

8. Verify the DR volume before failover
Check:

kubectl get volume.longhorn.io dr-test-data \
  -n longhorn-system \
  -o jsonpath='{.status.state}{" | "}{.status.robustness}{" | "}{.status.restoreRequired}{"\n"}'

Ideally:

detached | healthy | false

or attached/healthy once you mount it.

Then create the PV/PVC that points to:

dr-test-data

Do not accidentally point the PVC at the original Cluster A volume.

This distinction caused the confusing behavior in your previous test.

9. Test the DR volume independently
Before declaring failover successful, mount the restored volume to a temporary pod.

For example:

Cluster B
   |
   +-- dr-test-data
          |
          +-- PV
                |
                +-- test-data-dr
                       |
                       +-- temporary pod

Then:

kubectl exec -n dr-test <pod> -- \
  cat /data/test.txt

You must see:

DATA CREATED ON CLUSTER A

This proves:

Cluster A data
      ↓
Longhorn backup
      ↓
NFS
      ↓
Cluster B restore
      ↓
Cluster B Longhorn volume
      ↓
filesystem

Only after this works should you perform the application failover.

10. Deploy the application on Cluster B
Deploy the application manifests on Cluster B.

But make the application's PVC reference the restored DR volume.

For example:

Deployment
    ↓
PVC: test-data
    ↓
PV: dr-test-data-pv
    ↓
Longhorn volume: dr-test-data

Verify:

kubectl get pv
kubectl get pvc -n dr-test
kubectl get pods -n dr-test

You want:

PVC     Bound
Pod     Running

Then verify the application data:

kubectl exec -n dr-test <pod> -- \
  cat /usr/share/nginx/html/data/test.txt

It must contain the Cluster A data.

11. Perform Cluster A → Cluster B failover
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

On Cluster B:

Restore/create the DR volume from the latest available backup.
Wait for:
restoreRequired=false

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

12. Write new data on Cluster B
This step is essential.

Don't just verify the old A data.

Change the data:

kubectl exec -n dr-test <pod> -- \
  sh -c 'echo "DATA CREATED ON CLUSTER B AFTER FAILOVER" >> /usr/share/nginx/html/data/test.txt'

Verify:

kubectl exec -n dr-test <pod> -- \
  cat /usr/share/nginx/html/data/test.txt

You should now have:

DATA CREATED ON CLUSTER A
DATA CREATED ON CLUSTER B AFTER FAILOVER

13. Let Cluster B create a new backup
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

backup-a35af0bee6084662

for example.

And:

STATE=Completed

Do not start failback until this is confirmed.

14. Stop writes on Cluster B
Before failback:

kubectl scale deployment dr-test \
  -n dr-test \
  --replicas=0

Wait until:

kubectl get pods -n dr-test

shows no application pod writing to the volume.

Then confirm the final B backup completed.

15. On Cluster A, sync the backup target
Cluster A needs to discover the backup created by Cluster B.

Check:

kubectl get backupvolume.longhorn.io \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,LASTBACKUP:.status.lastBackupName,LASTBACKUPAT:.status.lastBackupAt,LASTSYNC:.status.lastSyncedAt'

The important point is:

Cluster B
    ↓
backup-a35...
    ↓
NFS
    ↓
Cluster A backup target
    ↓
backup-a35...

If Cluster A does not see the new backup, stop here.

Do not restore from the old backup.

Force/check backup target synchronization according to your Longhorn configuration and verify the new backup is visible.

16. Create a new failback volume on Cluster A
Do not reuse the old Cluster A volume blindly.

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

Wait until:

restoreRequired=false

and:

robustness=healthy

17. Verify the failback volume independently
This is the step we missed in the previous attempt.

Create a temporary PV/PVC/pod against:

dr-test-data-failback

Then:

kubectl exec -n dr-test <temporary-pod> -- \
  cat /data/test.txt

You must see:

DATA CREATED ON CLUSTER A
DATA CREATED ON CLUSTER B AFTER FAILOVER

If you see only the original A content, you are looking at the wrong volume or wrong backup.

Do not proceed until this check passes.

18. Point the Cluster A application at the failback volume
Once the restored volume has been independently verified:

Cluster A
   |
   +-- Application
          |
          +-- PVC
                |
                +-- PV
                      |
                      +-- dr-test-data-failback

The important thing is that the application's PVC must reference the restored failback PV, not:

pvc-c1997f6f-...

from the old Cluster A volume.

19. Start the application on Cluster A
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

20. Final failback verification
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

21. What you should monitor during every DR operation
For the Longhorn volume:

kubectl get volume.longhorn.io <volume> \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,STATE:.status.state,ROBUSTNESS:.status.robustness,RESTORE:.status.restoreRequired,LASTBACKUP:.status.lastBackup'

For backups:

kubectl get backups.longhorn.io \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,CREATED:.status.backupCreatedAt,STATE:.status.state,URL:.status.url'

For the backup volume:

kubectl get backupvolume.longhorn.io \
  -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,LASTBACKUP:.status.lastBackupName,LASTBACKUPAT:.status.lastBackupAt,LASTSYNC:.status.lastSyncedAt'

For Kubernetes storage:

kubectl get pv
kubectl get pvc -A

For applications:

kubectl get pods -A
kubectl get deployments -A

22. The most important rules
Deploy the application on both clusters.
Do not run the application actively against the same RWO data from both clusters.
Only one cluster should be the active writer.
Longhorn does not move your Deployment between clusters.
Longhorn provides the volume backup/restore/DR mechanism.
Always verify the backup ID before restoring.
Always verify state=Completed for the backup.
Always verify restoreRequired=false after restore.
Always verify the restored volume's contents before attaching it to the application.
Don't assume a PVC called test-data is using the volume you intended.
Always check the complete chain:
PVC
 ↓
PV
 ↓
CSI volumeHandle
 ↓
Longhorn Volume
 ↓
Longhorn Backup

During failover, stop Cluster A writes before activating B whenever possible.
During failback, stop Cluster B writes before activating A whenever possible.
Don't delete the old source volume until you've verified the restored volume.
Keep old backups according to your required RPO/RTO and retention policy.
Longhorn's own documentation emphasizes that its backup feature is intended for protecting volume data and supports cross-cluster DR; recurring backups are recommended for protected volumes. 
L
Longhorn
+1

The complete test in one view
             NORMAL OPERATION
             ================

        Cluster A                     Cluster B
       ┌───────────┐                ┌───────────┐
       │    APP    │                │    APP    │
       │  ACTIVE   │                │ STANDBY   │
       └─────┬─────┘                └─────┬─────┘
             │                            │
             ▼                            ▼
        Longhorn                     DR Volume
             │                            │
             └──────────┬─────────────────┘
                        ▼
                       NFS
                     BACKUPS


             FAILOVER A → B
             ==============

       Cluster A                     Cluster B
       STOP APP                     START APP
           X                              │
           │                              ▼
           │                         DR Volume
           │                              │
           └── backup ──► NFS ◄──────────┘
                                         │
                                         ▼
                                   B becomes ACTIVE


             FAILBACK B → A
             ==============

       Cluster A                     Cluster B
       START APP                     STOP APP
           ▲                              X
           │                              │
      New restored                        │
        volume                            │
           ▲                              │
           │                              │
           └──────── NFS ◄── backup ──────┘

                  A becomes ACTIVE

The key to avoiding the problem you encountered is that failover/failback is not simply "restore a Longhorn volume and expect the existing PVC to follow it." The PV/PVC must explicitly point at the restored Longhorn volume. Longhorn's system-restore documentation likewise notes that Longhorn does not restore existing Volumes together with their associated PV/PVC objects automatically. 
L
Longhorn

For your next clean test, I would use four unmistakable markers—A-INITIAL, B-AFTER-FAILOVER, A-AFTER-FAILBACK, and a final timestamp—and verify each one against the actual Longhorn volumeHandle before moving to the next stage. That will make it impossible to accidentally test the old volume again.
