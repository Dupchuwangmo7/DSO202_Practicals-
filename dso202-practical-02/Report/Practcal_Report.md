# DSO202 — Practical 2 Report

## Implementing Persistent Storage for a Stateful Application in Kubernetes

**Module:** DSO202
**Practical:** 2 — Implementing Persistent Storage for a Stateful Application in Kubernetes
**Date:** 7 September 2026


## 1. Objective

The objective of this practical was to understand and implement persistent storage for stateful applications in Kubernetes.

The practical focused on:

* Creating and using PersistentVolumes (PV) and PersistentVolumeClaims (PVC).
* Understanding static and dynamic volume provisioning.
* Configuring StorageClasses and reclaim policies.
* Understanding the difference between `Retain` and `Delete` reclaim policies.
* Observing how persistent volumes influence pod scheduling.
* Understanding the differences between Deployments and StatefulSets.
* Using headless Services to provide stable network identities.
* Using `volumeClaimTemplates` to provide individual persistent storage to StatefulSet replicas.
* Deploying PostgreSQL with persistent storage.
* Verifying that database data survives pod deletion and recreation.
* Understanding the difference between persistent storage and backups.



## 2. Environment

| Component               | Environment / Version              |
| ----------------------- | ---------------------------------- |
| Operating System        | macOS                              |
| Container Runtime       | Docker Desktop                     |
| Kubernetes Distribution | kind                               |
| Kubernetes Cluster      | `dso202-p2`                        |
| Kubernetes Version      | v1.36.1                            |
| kubectl                 | v1.36                              |
| Namespace               | `dso202-practical-02`              |
| Database                | PostgreSQL 18 Alpine               |
| Storage Provisioner     | `rancher.io/local-path`            |
| StorageClasses          | `standard`, `dso202-retain`        |
| Cluster Nodes           | 1 control-plane and 2 worker nodes |

The practical was implemented using Kubernetes YAML manifests and `kubectl` commands. Docker was used as the container runtime and kind was used to create the local Kubernetes cluster.


## 3. Procedure and Observations

### 3.1 Environment Preparation

The required tools were verified before starting the practical. Docker, kind and kubectl were available and working.

A directory for the practical's host-mounted storage was created:

```bash
mkdir -p /tmp/dso202-p2-storage
```

The Kubernetes environment was checked to ensure that no previous practical cluster would interfere with the exercises.

The kind cluster was then created using the provided configuration.

**Observation:**
The environment was successfully prepared for testing Kubernetes persistent storage.



### 3.2 Cluster and StorageClass Setup

A three-node kind cluster named `dso202-p2` was created.

The cluster and nodes were verified using:

```bash
kind get clusters
kubectl get nodes -o wide
```

The namespace `dso202-practical-02` was created along with the required resource quota and limit range.

The available StorageClasses were checked using:

```bash
kubectl get storageclass
```

The two important StorageClasses were:

| StorageClass    | Reclaim Policy | Binding Mode         | Volume Expansion |
| --------------- | -------------- | -------------------- | ---------------- |
| `standard`      | Delete         | WaitForFirstConsumer | Disabled         |
| `dso202-retain` | Retain         | WaitForFirstConsumer | Disabled         |

Both StorageClasses used the local-path provisioner.

**Observation:**
A StorageClass controls how storage is provisioned and what happens to the underlying volume when a PVC is deleted. The main difference between the two classes was their reclaim policy.



### 3.3 Static PersistentVolume

A static PersistentVolume named `pv-web-static` was created.

The PV used:

* 1 GiB capacity
* `ReadWriteOnce` access mode
* `Retain` reclaim policy
* `hostPath` storage
* Node affinity to a worker node

A PVC named `pvc-web-static` was created to request the storage.

The PV and PVC were checked using:

```bash
kubectl get pv,pvc
```

The PVC successfully became `Bound`.

A writer pod named `static-writer` was used to write a file named `ledger.txt` to the mounted volume.

The file was checked from inside the pod and from the host storage directory.

The pod was deleted and recreated. The previously created file was still present.

The PVC was then deleted. Because the PV used the `Retain` reclaim policy, the PV changed to:

```text
Released
```

The underlying data was not automatically removed.

The PV object was subsequently deleted and recreated along with a new PVC and pod. The existing data was still available.

**Observation:**
The storage data has a different lifecycle from the pod and Kubernetes storage objects. The `Retain` reclaim policy allows the underlying data to remain after the PVC or PV object is removed.


### 3.4 Dynamic Provisioning

A PVC named `dynamic-data` was created using the `standard` StorageClass.

Initially, the PVC showed:

```text
Pending
```

This was expected because the StorageClass used:

```text
WaitForFirstConsumer
```

A pod named `dynamic-writer` was then created to consume the PVC.

After the pod was scheduled, the PVC changed to:

```text
Bound
```

Kubernetes dynamically created a PV for the claim.

The dynamically created PV had a name similar to:

```text
pvc-<uid>
```

The storage was provisioned by the local-path provisioner on the node where the pod was scheduled.

The mounted filesystem was inspected using:

```bash
kubectl exec dynamic-writer -- df -h /data
```

The result showed the node's filesystem rather than a filesystem strictly limited to the requested PVC size.

A resize operation was attempted to increase the PVC from 1 GiB to 2 GiB. The operation was rejected because the StorageClass had:

```text
allowVolumeExpansion: false
```

The dynamic writer pod and PVC were then removed.

Because the `standard` StorageClass uses the `Delete` reclaim policy, the dynamically created PV and its underlying storage directory were removed.

**Observation:**
Dynamic provisioning reduces the need for manually creating PVs. The `WaitForFirstConsumer` mode delays provisioning until Kubernetes has a consumer pod and can determine suitable placement.

---

### 3.5 Shared Storage with a Deployment

A Deployment named `shared-writer` was created with three replicas using a single PVC.

The pods were inspected using:

```bash
kubectl get pods -o wide
```

The replicas were scheduled on the same node because the local-path volume was node-local and used `ReadWriteOnce`.

The replicas wrote data to the same:

```text
visitors.log
```

file.

One of the pods was deleted:

```bash
kubectl delete pod <pod-name>
```

The Deployment automatically created a replacement pod.

The replacement pod had a different generated name, but the shared volume and its existing data remained available.

**Observations:**

1. The storage location influenced where the pods could be scheduled.
2. All three replicas used the same shared data because they referenced the same PVC.
3. Deployment pod identities are not stable. When a pod is replaced, the new pod receives a new generated name.

This showed that Deployments treat replicas as interchangeable instances, unlike StatefulSets.

---

### 3.6 StatefulSet and Stable Storage

A headless Service named `webnote` was created with:

```yaml
clusterIP: None
```

A StatefulSet named `webnote` was deployed with three replicas.

The pods received stable ordinal names:

```text
webnote-0
webnote-1
webnote-2
```

The StatefulSet used `volumeClaimTemplates`, which created separate PVCs:

```text
content-webnote-0
content-webnote-1
content-webnote-2
```

Each StatefulSet pod therefore had its own persistent storage.

DNS resolution was tested from a client pod.

The headless Service resolved to the StatefulSet pod addresses.

An individual StatefulSet pod followed the DNS pattern:

```text
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

For example:

```text
webnote-1.webnote.dso202-practical-02.svc.cluster.local
```

A note was written to `webnote-0`.

The note was visible from `webnote-0` but was not present on the other StatefulSet replicas, demonstrating that each ordinal had its own storage.

The `webnote-1` pod was then deleted.

Kubernetes recreated the pod using the same stable name:

```text
webnote-1
```

The replacement pod reused its existing PVC and therefore retained the previous data.

Although the pod was recreated, its IP address could change.

**Observation:**
StatefulSets provide stable pod identity, stable storage and stable network identity. Each ordinal is associated with its own persistent volume claim.

---

### 3.7 StatefulSet Scaling and Rolling Updates

The `webnote` StatefulSet was scaled from three replicas to four:

```bash
kubectl scale statefulset webnote --replicas=4
```

A new PVC was automatically created:

```text
content-webnote-3
```

The StatefulSet was then scaled down to two replicas.

The higher ordinal pods were terminated first.

The PVCs remained because the StatefulSet was configured with:

```yaml
persistentVolumeClaimRetentionPolicy:
  whenScaled: Retain
```

The StatefulSet was scaled back to three replicas.

The `webnote-2` pod returned and reused its existing PVC and stored data.

A rolling update was also performed using a partitioned StatefulSet update. The partition was used to control which ordinal was updated first. After changing the partition, the remaining pods were updated according to StatefulSet update behaviour.

The StatefulSet was deleted while its PVCs were retained because the StatefulSet was configured with:

```yaml
whenDeleted: Retain
```

The StatefulSet was then recreated and the existing PVCs were reused.

**Observation:**
StatefulSet ordinals create a stable relationship between a pod and its persistent storage. Scaling down does not necessarily destroy the data belonging to the removed ordinal.

---

### 3.8 PostgreSQL StatefulSet

A Kubernetes Secret named:

```text
postgres-credentials
```

was created to store the PostgreSQL credentials.

Two Services were configured:

* `postgres` — normal ClusterIP Service for database connections.
* `postgres-headless` — headless Service used for StatefulSet network identity.

A PostgreSQL 18 Alpine StatefulSet was deployed.

The PostgreSQL persistent volume was mounted at:

```text
/var/lib/postgresql
```

The PostgreSQL 18 data directory was located underneath this directory.

A readiness probe was configured so that the PostgreSQL pod would only receive Service traffic after the database was ready.

The database PVC was configured with:

* Capacity: 2 GiB
* Access mode: ReadWriteOnce
* StorageClass: `dso202-retain`
* Reclaim policy: Retain

A `tasks` table was created and populated with three records.

| ID | Task                 |
| -- | -------------------- |
| 1  | Complete Practical 2 |
| 2  | Read Unit II notes   |
| 3  | Draft the report     |

The records were verified using:

```sql
SELECT count(*) FROM tasks;
```

The result was:

```text
3
```

The PostgreSQL pod `postgres-0` was then deleted.

Kubernetes recreated the pod. After the new pod became Ready, the database was accessed again and the same query was executed.

The result remained:

```text
3
```

This demonstrated that the database data survived the deletion and recreation of the PostgreSQL pod.

**Observation:**
The PostgreSQL data was stored on the persistent volume rather than being dependent on the lifetime of the container or pod. The PVC was reused by the replacement pod.

---

### 3.9 Evidence Collection and Cleanup

Final Kubernetes resources were recorded using commands such as:

```bash
kubectl get all -o wide > evidence/final-state-all.txt
kubectl get pv,pvc,storageclass -o wide > evidence/final-state-storage.txt
kubectl get events --sort-by=.lastTimestamp > evidence/final-state-events.txt
```

A PostgreSQL database dump was also created:

```bash
kubectl exec postgres-0 -- pg_dump -U taskuser -d tasktracker > evidence/tasktracker-dump.sql
```

The workloads and PVCs were then removed during cleanup.

The resulting storage behaviour demonstrated that:

* Volumes using the `Delete` reclaim policy were removed automatically.
* Volumes using `Retain` became `Released`.
* Retained underlying data was not automatically deleted.
* Static hostPath data remained on the host.
* Dynamic local-path data stored inside kind nodes disappeared when the nodes were removed with the cluster.

**Observation:**
Persistent storage and backups are different concepts. A retained PV can preserve data, but an external database dump is required for an independent backup and recovery mechanism.


## 4. Analysis

The practical demonstrated how Kubernetes separates the lifecycle of compute resources from the lifecycle of application data. Pods are temporary resources, while PVs and PVCs allow data to survive pod replacement.

Static provisioning requires a PV to be created manually before a PVC can use it. Dynamic provisioning is more automated because a StorageClass allows Kubernetes to create a suitable PV when a PVC is consumed.

The `WaitForFirstConsumer` binding mode was important when using local storage. The PVC initially remained `Pending` because Kubernetes waited until a pod required the volume and could therefore determine the appropriate node.

The reclaim policy also had a significant effect on data lifecycle. The `Delete` policy automatically removed dynamically created storage when its PVC was deleted, while the `Retain` policy preserved the underlying storage after the PVC was removed.

The Deployment and StatefulSet experiments demonstrated the difference between stateless and stateful workload management. Deployment replicas are interchangeable and receive new identities when replaced. StatefulSet replicas have stable ordinal names and are associated with stable PVCs.

The headless Service provided stable DNS identities for the StatefulSet. This is particularly important for distributed or clustered applications where individual instances need to discover and communicate with each other.

The PostgreSQL experiment demonstrated the practical importance of persistent storage. The database records remained available after `postgres-0` was deleted and recreated because the new pod reused the same persistent volume claim.

The local-path provisioner was useful for demonstrating Kubernetes storage concepts, but it also has limitations. Storage is tied to the node, and the requested PVC capacity is not necessarily enforced as a separate filesystem size. Production environments generally require more robust CSI-based storage, replication, backup and recovery mechanisms.


## 5. Reflection

This practical improved my understanding of how Kubernetes manages persistent storage for stateful applications.

One of the most important concepts I learned was the relationship between a PVC, PV and StorageClass. A PVC represents an application's request for storage, while the PV provides the storage resource and the StorageClass defines how that storage is provisioned.

The `Pending` state of the dynamic PVC was also useful because it showed that not every Pending PVC represents an error. With `WaitForFirstConsumer`, Kubernetes intentionally waits for a pod before completing the volume provisioning process.

The failed PVC resize attempt demonstrated that storage expansion depends on StorageClass and provisioner capabilities. This helped me understand that Kubernetes does not automatically guarantee that every storage volume can be resized.

The StatefulSet exercise was particularly useful because it showed how Kubernetes maintains stable identities for stateful applications. When `webnote-1` was deleted, Kubernetes recreated the same ordinal and reused its existing storage instead of assigning it a completely new identity.

The PostgreSQL exercise connected the concepts to a realistic application. The database records remained after pod deletion because the data was stored on persistent storage rather than inside the temporary pod filesystem.

I also learned that persistent storage alone is not a complete backup solution. A retained volume protects against certain resource deletion scenarios, but an independent database backup such as the PostgreSQL dump is still necessary for reliable disaster recovery.

Overall, the practical gave me a clearer understanding of persistent storage, storage provisioning, reclaim policies, StatefulSets, stable networking and database persistence in Kubernetes.

Sorry machedsuch klass came rapshour late I to let me take bilingualized on hold down retrived your payments first always recognized that they always proactive say in night that they always remember this is placed anyways Domingo Shitmost totally opposite back questions he knows about you and he let you go drunks you can always add every single person I have ever dropped two data across on who do you think I will run to one two and following the vote and no way do you haveon quick drinks where this quid does not do you want zero and it's not that shemeans that the chicken to gym is heads I mean talking or I never generate so much since or sitting on research box technical chickens came up and body moderate semester I don't think I can work celebrate horsewocking matchu was a nice breath yeah you don't bounce much to your next time ante she said new texas I don't like the sexualenergy the major said you know like I knew and how we dare and push put on the day and last video am jitu go to do you melted three video then since locks ten dollars to report the common match one so no chicken part condition discuss on gold studies and the same draw in the chamber dress I know that the truth you three guys more contact could you go so on the matter my domages for four ahead of twice if you are the best around a best compound with best look make up