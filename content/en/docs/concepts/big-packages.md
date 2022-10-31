---
title: Big Packages
draft: false
images: []
weight: 900
toc: true
---

The most straightforward way of using the Package Operator APIs `ObjectDeployment` and `ObjectSet` is to define objects inline directly when creating an instance of these APIs.

etcd - the default Kubernetes database - has an object size limit of 1 MiB ([etcd <=v3.2](https://etcd.io/docs/v3.2/dev-guide/limit/)) or 1.5 MiB ([etcd >v3.2](https://etcd.io/docs/v3.3/dev-guide/limit/)).

Building packages containing multiple large objects, like CustomResourceDefinitions, or just contain a large number of objects, might run into these limits, when defining objects inline.

## Slices API

To get around this limitation, Package Operator allows offloading big objects into an auxiliary API `ObjectSlice`. Instead of all objects being defined inline in the parent `ObjectDeployment` or `ObjectSet` one or multiple `ObjectSlice`s can be specified.

In contrast to `ObjectDeployment`s, `ObjectSlice`s are immutable.  
When updating an `ObjectDeployment` a new `ObjectSlice` needs to be created, which contains the desired changes.
Only when referencing this new slice instead of the current slice from the ObjectDeployment, will the change be applied.

{{< alert text="When using the `Package` API, Package Operator will automatically split packages that reach a certain limit by using the `ObjectSlice` API." />}}

## Example

```yaml
apiVersion: package-operator.run/v1alpha1
kind: ObjectSet
metadata:
  name: example
  namespace: default
spec:
  availabilityProbes: []
  phases:
  - name: phase-1
    objects:
    - object: # inline defined object
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: example-deployment
    slices: # objects referenced from ObjectSlice API.
    - example-slice-001
    - example-slice-002
    - example-slice-003
---
apiVersion: package-operator.run/v1alpha1
kind: ObjectSlice
metadata:
  name: example-slice-001
  namespace: default
spec:
  objects:
  - object: # defined object
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: example-deployment-1
# ...
```
