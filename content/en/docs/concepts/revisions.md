---
title: Revisions
draft: false
images: []
weight: 700
toc: true
mermaid: true
---

```mermaid
stateDiagram-v2
direction LR
  state "Revision 1" as rev1
  state "Revision 2" as rev2
  state "Revision 3" as rev3
  [*] --> rev1
  rev1 --> rev2
  rev2 --> rev3
  rev3 --> [*]
```

Revisions are iterations and changes of a deployment over time. To support zero-downtime
deployments, even if something goes wrong, Package Operator can manage multiple
active revisions at the same time. This strategy is also often referred to as
"A/B deployment" or "canary deployment".

A revision is represented by the `ObjectSet`/`ClusterObjectSet` APIs.

While revision are ordered on a time axis, Package Operator makes no assumptions
on the contents of each revisions. This means that Revision 3 could contain the
same spec as Revision 1, rolling back changes introduced by Revision 2.

## Revision Lifecycle

```mermaid
stateDiagram-v2
direction LR
  state "Not Ready" as not_ready
  [*] --> Pending
  Pending --> Available
  Available --> not_ready
  Pending --> not_ready
  not_ready --> Available
  Available --> Archived
  not_ready --> Archived
  Archived --> [*]
```

- **Pending**\
  Intermediate state before the controller posted its first update.
- **Available**\
  All availability probes are successful.
- **Not Ready**\
  One or more availability probes are unsuccessful.
- **Archived**\
  (Cluster)ObjectSet is shutdown and only acts as a revision tombstone for rollbacks.

In addition to these major lifecycle states, (Cluster)ObjectSets may be **Paused**,
stopping reconciliation, while still reporting status.
This can be useful for testing and debugging.

## Rollout Handling

Depending on the contents of the new revision, objects are eventually either:

- **Patched**\
  If the object still part of the new revision, it will be handed over to the next
  revision.
- **Deleted**\
  If the object is not part of the new revision, it will be deleted when the old
  revision is archived.

### In-Place Updates

```mermaid
flowchart LR
  subgraph Revision 1
    deploy1["Deployment <b>my-deployment</b><br>image: my-image:rev1"]
  end
  subgraph Revision 2
    deploy2["Deployment <b>my-deployment</b><br>image: my-image:rev2"]
  end
  deploy1--patched object-->deploy2
```

"In-Place" updates happen when a new revision contains an object with the same name
and type as the previous revision. Objects are handed over to a new revision and
patched as needed.

When all objects have been handed over to a new revision, the previous revision
is automatically **archived**.

{{< alert text=`Updating in-place may not provide any safety net.<br>If the Update
fails, your deployment may face downtime.` />}}

### A/B Updates

```mermaid
flowchart LR
  subgraph Revision 1
    deploy1["Deployment <b>my-deployment-v1</b><br>image: my-image:rev1"]
  end
  subgraph Revision 2
    deploy2["Deployment <b>my-deployment-v2</b><br>image: my-image:rev2"]
  end
  deploy1-. indirect successor .->deploy2
```

A/B updates happen when a new revision does not contain an object with the same
name and type as a previous revision. A new object is created in the new revision
without affecting the old object.

The old revision is only archived when the new revision has completely finished
its rollout and is "Available".

### Intermediate (Failed) Revisions

```mermaid
flowchart LR
  subgraph rev1[Revision 1]
    cm1["ConfigMap <b>my-config</b><br>data: {key: value-v1}"]
    deploy1["Deployment <b>my-deployment-v1</b><br>image: my-image:rev1"]
  end
  subgraph rev2[Revision 2]
    cm2["ConfigMap <b>my-config</b><br>data: {key: value-v2}"]
    deploy2["Deployment <b>my-deployment-v2</b><br>image: my-image:does-not-exist"]
  end
  subgraph rev3[Revision 3]
    cm3["ConfigMap <b>my-config</b><br>data: {key: value-v3}"]
    deploy3["Deployment <b>my-deployment-v3</b><br>image: my-image:rev3"]
  end
  style rev2 fill:#ffbabd,stroke:#9f7b7d
  deploy1-->deploy2
  deploy2-->deploy3
  cm1-.->cm2
  cm2-.->cm3
```

Under normal circumstances at max 2 Revisions can be active during rollout. An old
and a new revision.

If a revision fails to become available due to e.g. misconfiguration and a new revision
supersedes it, multiple intermediate revisions might be active until the latest
revision becomes available.

Intermediate revisions will only be cleaned up if:

- Latest revision becomes available
- Revision is not reconciling any objects anymore
- Latest revision is not containing any object still actively reconciled by an intermediate

{{< alert text=`This behavior is necessary, so Package Operator can ensure the
safe handover of objects between revisions.` />}}

In the example above, the ConfigMap "my-config" is handed over from revision 1 to
revision 2 and in the end to revision 3.

As soon as revision 3 takes ownership of the ConfigMap, the failed intermediate
revision 2 can be archived, as "my-deployment-v2" no longer exists in revision 3
and is thus safe to delete.

## Internals

### ObjectSet to ObjectSet Handover

```yaml
apiVersion: package-operator.run/v1alpha1
kind: ObjectSet
metadata:
  name: v1
spec:
  phases:
  - name: phase-1
    objects: [{name: child-1}]
  - name: phase-2
    objects: [{name: child-2}]
---
apiVersion: package-operator.run/v1alpha1
kind: ObjectSet
metadata:
  name: v2
spec:
  phases:
  - name: phase-1
    objects: [{name: child-1}]
  - name: phase-2
    objects: [{name: child-2}]
  previous:
  - name: v1
    kind: ObjectSet
    group: package-operator.run
```

```mermaid
sequenceDiagram
autonumber
  participant v1 as ObjectSet v1 Reconciler
  participant api as kube-apiserver
  participant v2 as ObjectSet v2 Reconciler

  loop Reconciliation
    v1->>api: reconcile/update child-1 .spec
    v1->>api: reconcile/update child-2 .spec
  end

  Note over v2: phase-1 starts
  v2->>api: Take ownership of child-1 and<br> reconcile/update .spec
  par ObjectSet v1 Reconciler
    api-)v1: child-1 Update event
    activate v1
    v1->>api: reconcile/update child-2 .spec
    deactivate v1
  and ObjectSet v2 Reconciler
    api-)v2: child-1 Update event
    activate v2
    v2->>api: reconcile/update child-1 .spec
    deactivate v2
  end

  Note over v1,v2: Until phase-1 completes
  loop Reconciliation
    v2->>api: reconcile/update child-1 .spec
    v1->>api: reconcile/update child-2 .spec
  end

  Note over v2: phase-2 starts
  v2->>api: reconcile/update child-1 .spec
  activate v2
  v2->>api: Take ownership of child-2 and<br> reconcile/update .spec
  par ObjectSet v1 Reconciler
    api-)+v1: child-1 Update event
    deactivate v1
  and ObjectSet v2 Reconciler
    api-)v2: child-1 Update event
    activate v2
    v2->>api: reconcile/update child-1 .spec
    v2->>api: reconcile/update child-2 .spec
    deactivate v2
  end
```
