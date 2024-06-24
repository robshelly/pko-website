---
title: Status Probes
draft: false
images: []
weight: 400
toc: true
---

Status probes define how Package Operator judges the
**Availability** of objects and is reporting status.

Defining **Availability** will depend on the specific application that is deployed.\
In general, availability should reflect the health of the complete application bundle,
so Package Operator can check whether it's safe to roll over to a new revision.

Package Operator does not provide any default probes and leaves it to the author
of a package to configure probing explicitly. This ensures that probing of packages
stays consistent throughout different Package Operator releases and allows package
authors to tweak probing to their specific requirements.

## Probe Spec

Probes typically consist of two parts.\
A selector specifying what objects to apply a probe to and a list of probes to check.

All available probing declarations can be found in [API Reference - ObjectSetProbe](/docs/api_reference/package-operator-api#objectsetprobe).

## Examples

### Deployment

```yaml
selector:
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

### StatefulSet

```yaml
selector:
  kind:
    group: apps
    kind: StatefulSet
probes:
- condition:
    type: Available
    status: "True"
- fieldsEqual:
    fieldA: .status.updatedReplicas
    fieldB: .status.replicas
```

### CustomResourceDefinition

```yaml
selector:
  kind:
    group: apiextensions.k8s.io
    kind: CustomResourceDefinition
probes:
- condition:
    type: Established
    status: "True"
```

### OpenShift Route

```yaml
selector:
  kind:
    group: route.openshift.io
    kind: Route
probes:
- cel:
    message: not all ingress points are reporting ready
    rule: self.status.ingress.all(i, i.conditions.all(c, c.type == "Ready" && c.status == "True"))
```
