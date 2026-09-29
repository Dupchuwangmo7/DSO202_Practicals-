
# DSO202 Practical 2 Report

## Implementing Persistent Storage for a Stateful Application in Kubernetes

**Student:** Dupchu Wangmo  
**Module:** DSO202  
**Practical:** Practical 2  
**Date:** 6 September 2026  
**Platform:** macOS, Docker Desktop, kind Kubernetes Cluster


# 1. Objective

The objective of this practical was to deploy and manage a stateful application using Kubernetes. The practical focused on StatefulSets, PersistentVolumeClaims (PVCs), StorageClasses, Services, persistent storage, StatefulSet scaling behaviour, Pod management policies, custom Pod ordinals, and PostgreSQL backup and recovery.

The practical also aimed to demonstrate how persistent storage allows application data to survive Pod replacement and how Kubernetes resources can be inspected and troubleshot using `kubectl`.



# 2. Environment

The practical was completed using the following environment:

| Component | Configuration |
|---|---|
| Operating System | macOS |
| Container Runtime | Docker Desktop |
| Kubernetes Distribution | kind |
| Kubernetes Version | v1.36.1 |
| Cluster | kind |
| Node | kind-control-plane |
| Namespace | `dso202-practical-02` |
| Web Application | NGINX |
| NGINX Image | `nginx:1.30-alpine` |
| PostgreSQL Image | `postgres:18-alpine` |
| Storage Provisioner | `rancher.io/local-path` |
| PostgreSQL Database | `tasktracker` |

The Kubernetes cluster was verified using:

```bash
kubectl get nodes
````

The node was successfully reported as:

```text
kind-control-plane   Ready   control-plane   v1.36.1
```



# 3. Procedure and Observations

## 3.1 Namespace and StorageClass

The namespace for the practical was created and selected:

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=dso202-practical-02
```

The required StorageClass was also applied.

The cluster used the `rancher.io/local-path` provisioner for dynamic persistent volume provisioning.

The PVCs were successfully created and reached the `Bound` state.

This demonstrated that Kubernetes can dynamically provision storage when a PersistentVolumeClaim is created.



## 3.2 StatefulSet Deployment

The `webnote` application was deployed using a StatefulSet with three replicas.

The Pods created were:

```text
webnote-0
webnote-1
webnote-2
```

Each Pod received its own PVC:

```text
content-webnote-0
content-webnote-1
content-webnote-2
```

The PVCs were successfully bound to persistent volumes.

The StatefulSet used a headless Service named:

```text
webnote
```

The headless Service provides stable network identities for StatefulSet Pods.

The application also used an init container to initialise content in the mounted persistent storage.



# 4. Extension 1 — PVC Retention Policy

The StatefulSet was configured with:

```yaml
persistentVolumeClaimRetentionPolicy:
  whenDeleted: Retain
  whenScaled: Delete
```

This configuration controls what happens to PVCs during StatefulSet lifecycle operations.

The StatefulSet was initially running three replicas:

```text
webnote-0
webnote-1
webnote-2
```

The StatefulSet was then scaled down:

```bash
kubectl scale statefulset webnote --replicas=1
```

After scaling down, only:

```text
webnote-0
```

remained.

Because:

```yaml
whenScaled: Delete
```

was configured, the PVCs associated with the removed replicas were deleted.

The StatefulSet was then scaled back to three replicas, causing Kubernetes to create the required Pods and PVCs again.

The important observation was that the retention behaviour is different depending on the lifecycle event:

| Event                   | Policy   | Result                                |
| ----------------------- | -------- | ------------------------------------- |
| StatefulSet deleted     | `Retain` | PVCs are retained                     |
| StatefulSet scaled down | `Delete` | PVCs for removed replicas are deleted |

This demonstrated how PVC lifecycle can be controlled using StatefulSet retention policies.



# 5. Extension 2 — Parallel Pod Management

The original StatefulSet used:

```yaml
podManagementPolicy: OrderedReady
```

With `OrderedReady`, StatefulSet Pods are created and managed in ordinal order.

The extension investigated the alternative:

```yaml
podManagementPolicy: Parallel
```

The difference between the two policies is:

| Policy         | Behaviour                                                                         |
| -------------- | --------------------------------------------------------------------------------- |
| `OrderedReady` | Pods are managed sequentially and readiness affects progression                   |
| `Parallel`     | Pods can be created and managed without waiting for previous Pods to become Ready |

The `Parallel` policy is useful when replicas are independent and do not require a particular startup order.

This extension demonstrated that StatefulSet Pod management behaviour can affect how quickly replicas are created and updated.

---

# 6. Extension 3 — StatefulSet Ordinals

Kubernetes support for custom StatefulSet ordinals was checked using:

```bash
kubectl explain statefulset.spec.ordinals
```

The cluster supported the `ordinals` field.

The StatefulSet was configured with:

```yaml
replicas: 3

ordinals:
  start: 2
```

This caused the StatefulSet to use the ordinal numbers:

```text
2
3
4
```

Therefore, the Pods were:

```text
webnote-2
webnote-3
webnote-4
```

The corresponding PVCs were:

```text
content-webnote-2
content-webnote-3
content-webnote-4
```

The ordinal range can be calculated as:

```text
start = 2
replicas = 3

2, 3, 4
```

This demonstrated that StatefulSet replicas do not necessarily have to start at ordinal `0`.

Custom ordinals can be useful for migration scenarios or when maintaining specific identity ranges for stateful workloads.

---

# 7. Extension 4 — PostgreSQL Backup and Recovery

A PostgreSQL StatefulSet was deployed using:

```text
postgres:18-alpine
```

The PostgreSQL database was:

```text
tasktracker
```

The database used a persistent volume through:

```text
data-postgres-0
```

The PVC was successfully bound:

```text
data-postgres-0   Bound   2Gi   RWO   dso202-retain
```

## 7.1 Creating the Tasks Table

The following table was created:

```sql
CREATE TABLE tasks (
    id serial PRIMARY KEY,
    title text NOT NULL,
    done boolean NOT NULL DEFAULT false,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

Three records were inserted:

```text
Complete Practical 2
Read Unit II notes
Draft the report
```

The data was verified using:

```bash
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker \
  -c "SELECT id, title, done FROM tasks ORDER BY id;"
```

The database contained the expected three records.



## 7.2 Creating the Backup

A PostgreSQL logical backup was created using:

```bash
mkdir -p evidence

kubectl exec postgres-0 -- pg_dump -U taskuser -d tasktracker \
  > evidence/tasktracker-dump.sql
```

The generated dump was checked using:

```bash
ls -lh evidence/tasktracker-dump.sql
```

The dump provided an independent backup of the database.



## 7.3 Simulating Data Loss

The `tasks` table was removed:

```bash
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker \
  -c "DROP TABLE tasks;"
```

The database was then checked:

```bash
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker \
  -c "\dt"
```

The `tasks` table was no longer present.

This simulated accidental data loss.



## 7.4 Restoring the Database

The dump file was copied into the PostgreSQL Pod:

```bash
kubectl cp evidence/tasktracker-dump.sql \
  postgres-0:/tmp/tasktracker-dump.sql
```

The backup was restored:

```bash
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker \
  -f /tmp/tasktracker-dump.sql
```

The table and data were then verified:

```bash
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker \
  -c "SELECT id, title, done FROM tasks ORDER BY id;"
```

The three original records were successfully restored.



## 7.5 Testing Persistence After Pod Deletion

The PostgreSQL Pod was deleted:

```bash
kubectl delete pod postgres-0
```

Because PostgreSQL was managed by a StatefulSet, Kubernetes recreated the Pod.

The Pod was monitored using:

```bash
kubectl get pod postgres-0 -w
```

Once the Pod became Ready, the database was checked again:

```bash
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker \
  -c "SELECT id, title, done FROM tasks ORDER BY id;"
```

The data remained available.

This proved that the database data was stored on persistent storage rather than only inside the temporary Pod filesystem.



# 8. Troubleshooting

During the PostgreSQL extension, the PostgreSQL Pod initially remained in:

```text
ContainerCreating
```

Attempts to execute commands inside the Pod resulted in:

```text
error: unable to upgrade connection: container not found ("postgres")
```

The container name was checked using:

```bash
kubectl get pod postgres-0 \
  -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

The output confirmed that the container was actually named:

```text
postgres
```

The Pod description was then inspected:

```bash
kubectl describe pod postgres-0
```

The Events section showed:

```text
Pulling image "postgres:18-alpine"
```

This showed that the container had not started because the image was still being pulled.

The PVC was also checked:

```bash
kubectl get pvc
```

The PVC was already:

```text
data-postgres-0   Bound
```

Therefore, the problem was not related to persistent storage.

After the image was successfully pulled, the Pod changed to:

```text
postgres-0   1/1   Running
```

The `kubectl exec` commands then worked correctly.


# 9. Reviewed Questions and Answers

## Question 1: Why is a StatefulSet more suitable than a Deployment for a stateful application?

A StatefulSet provides stable Pod identities, stable network identities, and persistent storage associated with individual replicas.

A Deployment normally treats Pods as interchangeable, while stateful applications often require stable identities and storage.

**Answer:** StatefulSet is more suitable because stateful applications require stable identities and persistent storage for their replicas.



## Question 2: What is the purpose of a headless Service?

A headless Service provides stable DNS-based network identities for StatefulSet Pods.

For example:

```text
webnote-0
webnote-1
webnote-2
```

can have stable identities through the StatefulSet and its headless Service.

**Answer:** A headless Service allows StatefulSet Pods to be discovered through stable DNS identities.



## Question 3: Why does each StatefulSet replica receive a different PVC?

Stateful applications may require separate storage for each replica.

The StatefulSet uses `volumeClaimTemplates` to automatically create a PVC for each replica.

**Answer:** Each replica receives its own PVC so its persistent data remains separate from other replicas.



## Question 4: What is the difference between `Retain` and `Delete`?

`Retain` keeps the PVC when the relevant StatefulSet lifecycle event occurs.

`Delete` allows Kubernetes to delete the PVC automatically.

In this practical:

```yaml
whenDeleted: Retain
whenScaled: Delete
```

means that PVCs are retained when the StatefulSet is deleted, while PVCs belonging to replicas removed during scale-down are deleted.

**Answer:** `Retain` preserves the PVC, while `Delete` allows Kubernetes to remove it automatically.



## Question 5: What is the difference between `OrderedReady` and `Parallel`?

`OrderedReady` manages Pods sequentially according to their ordinal numbers and waits for readiness.

`Parallel` allows Pods to be managed without requiring the previous ordinal Pod to become Ready first.

**Answer:** `OrderedReady` provides sequential management, while `Parallel` allows concurrent management of replicas.



## Question 6: What does `ordinals.start: 2` do?

It changes the first ordinal number used by the StatefulSet.

With:

```yaml
replicas: 3

ordinals:
  start: 2
```

the replicas receive ordinals:

```text
2, 3, 4
```

instead of:

```text
0, 1, 2
```

**Answer:** It makes StatefulSet replica numbering start at `2`.


## Question 7: Why did `kubectl exec` initially fail for PostgreSQL?

The PostgreSQL Pod was still in:

```text
ContainerCreating
```

The container had not started yet, so Kubernetes could not establish an exec session.

The Pod events showed:

```text
Pulling image "postgres:18-alpine"
```

**Answer:** `kubectl exec` failed because the PostgreSQL container had not started while Kubernetes was pulling the image.


## Question 8: Why did the `tasks` table not exist initially?

The PostgreSQL database was created successfully, but the application table had not yet been created.

The command:

```bash
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker -c "\dt"
```

returned:

```text
Did not find any tables.
```

The table was then created manually.

**Answer:** The database was initially empty because creating a database does not automatically create application tables.



## Question 9: Why did the PostgreSQL data survive Pod deletion?

The PostgreSQL data directory was mounted from the PVC:

```text
data-postgres-0
```

Deleting the Pod did not delete the PVC.

When Kubernetes recreated `postgres-0`, it mounted the same persistent volume.

**Answer:** The data survived because it was stored on persistent storage rather than inside the temporary Pod filesystem.



## Question 10: Why is `pg_dump` useful when persistent storage is already available?

Persistent storage protects data from Pod replacement, but it is not a complete backup strategy.

Data can still be lost through accidental deletion, corruption, application errors, or other failures.

`pg_dump` creates a separate logical backup that can be restored later.

**Answer:** Persistent storage provides data durability, while `pg_dump` provides an independent backup and recovery mechanism.


# 10. Analysis

The practical demonstrated the importance of using StatefulSets for applications that require stable identities and persistent storage.

The `webnote` StatefulSet showed that each replica can have its own persistent volume. The PVC retention extension demonstrated that storage lifecycle can be controlled according to the requirements of the application.

The `Parallel` extension showed that StatefulSets do not always need to start Pods sequentially. When replicas are independent, parallel management can reduce waiting time.

The `ordinals.start` extension demonstrated that StatefulSet identities can be customised. Starting from ordinal `2` resulted in:

```text
webnote-2
webnote-3
webnote-4
```

The PostgreSQL experiment demonstrated the difference between Pod persistence and data persistence. A Pod can be deleted and recreated while the data remains available because the data is stored in the persistent volume.

The backup and recovery test also showed that persistent storage should not be considered a replacement for backups. The `pg_dump` file provided an independent recovery mechanism.



# 11. Reflection

One of the main troubleshooting issues encountered during the practical was the PostgreSQL Pod remaining in `ContainerCreating`.

Initially, `kubectl exec` was attempted, but the command failed because the container had not started. Instead of repeatedly trying the same command, the Pod status and events were checked using:

```bash
kubectl describe pod postgres-0
```

The Events section showed:

```text
Pulling image "postgres:18-alpine"
```

This helped identify that Kubernetes was still downloading the PostgreSQL image.

The PVC was also checked and was already in the `Bound` state. Therefore, the problem was correctly identified as a container image startup issue rather than a storage issue.

Another observation was that the PostgreSQL database initially contained no `tasks` table. The `\dt` command clearly showed that the database was empty, so the required table and test data were created before generating the backup.

If I repeated the practical, I would check Pod status and Kubernetes events before attempting `kubectl exec`. I would also verify the database schema and records before starting the backup and recovery experiment.

The practical improved my understanding of how StatefulSets, Pods, PVCs, StorageClasses, and persistent data are related. It also demonstrated that Kubernetes troubleshooting should be based on status information and events rather than repeatedly trying commands without checking the underlying problem.

One area I would like to understand further is how StatefulSet persistent storage and retention policies should be designed in a larger production environment using distributed storage across multiple Kubernetes nodes.


# 12. Conclusion

This practical successfully demonstrated how Kubernetes can be used to deploy and manage stateful applications.

The StatefulSet provided stable Pod identities, while PersistentVolumeClaims provided persistent storage for application and database data. The headless Service provided stable networking for StatefulSet replicas.

The extensions demonstrated:

1. PVC retention policies.
2. Parallel StatefulSet Pod management.
3. Custom StatefulSet ordinal numbering.
4. PostgreSQL backup and recovery using `pg_dump`.

The PostgreSQL experiment was particularly useful because it demonstrated that deleting a Pod does not necessarily result in data loss when persistent storage is correctly configured.

Overall, the practical showed that Pods are temporary compute resources, while persistent storage provides durability for important stateful application data. It also demonstrated that backups such as `pg_dump` are still necessary for reliable disaster recovery.







