---
title: Phases
draft: false
images: []
weight: 600
toc: true
mermaid: true
---

Phases are part of `ObjectSets` and `ClusterObjectSets` and order rollout and teardown of individual objects within a package revision to ensure repeatable and deterministic behavior.

When an `ObjectSet` is being reconciled, it will go through every specified phase in order. First creating or patching all objects contained within that phase and then probing them for availability.

Only when all objects within a phase are passing their availability probes, will the `ObjectSet` continue with the next phase, until all phases have been serviced.

{{< alert text="Order of objects within a phase is not guaranteed. Use multiple phases, if order-of-operations is important." />}}

## Teardown

When deleting an `ObjectSet` its phases are deleted in reversed. Waiting for objects to be gone from the kube-apiserver after all finalizers have been serviced, before continuing with the next phase.

## Example

Use multiple phases to ensure prerequisites are setup, before they are required.

In this example, a Namespace is created first, as RBAC roles and the deployment need to be within that namespace.

RBAC and a CustomResourceDefinition have to be applied before the deployment is started to prevent the deployment from accessing APIs that are either missing or it is not yet allowed to work with.

```yaml
phases:
- name: namespace
  objects:
  - object: {apiVersion: v1, kind: Namespace}
- name: crds-and-rbac
  objects:
  - object: {apiVersion: v1, kind: ServiceAccount}
  - object: {apiVersion: rbac.authorization.k8s.io/v1, kind: Role}
  - object: {apiVersion: rbac.authorization.k8s.io/v1, kind: RoleBinding}
  - object: {apiVersion: apiextensions.k8s.io/v1, kind: CustomResourceDefinition}
- name: deploy
  objects:
  - object: {apiVersion: apps/v1, kind: Deployment}

availabilityProbes:
- selector:
    kind:
      group: apiextensions.k8s.io
      kind: CustomResourceDefinition
  probes:
  - condition:
      type: Established
      status: "True"
- selector:
    kind:
      group: apps
      kind: Deployment
  probes:
  - condition:
      type: Available
      status: "True"
  - fieldsEqual:
      fieldA: .status.updatedReplicas
      fieldB: .status.replicas
```
