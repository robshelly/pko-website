---
title: Handover Process
draft: false
images: []
weight: 700
toc: true
mermaid: true
---

## (Cluster)ObjectSet Lifecycle

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

- **Pending**  
  Intermediate state before the controller posted it's first update.
- **Available**  
  All availability probes are successful.
- **Not Ready**  
  One or more availability probes are unsuccessful.
- **Archived**  
  (Cluster)ObjectSet is shutdown and only acts as a revision tombstone for rollbacks.

Additionally to these major lifecycle states, (Cluster)ObjectSets may be **Paused**, stopping reconciliation, while still reporting status.
This can be useful for testing and debugging.

## ObjectSet to ObjectSet

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
