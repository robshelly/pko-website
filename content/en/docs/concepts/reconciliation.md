---
title: Object Reconciliation
draft: false
images: []
weight: 200
toc: true
---

Package Operator is watching and if needed reconciling all objects under management.  
This page describes in detail how individual objects are updated.

Ordering of multiple objects is described on the [Phases page](/docs/concepts/phases).
How object status is interpreted is described further on the [Probes page](/docs/concepts/probes).

Object update rules:
- specified fields **MUST** always be reconciled to reset changes by other users
- additional labels and annotations for e.g. cache-control **MUST** be respected
- unspecified fields **MAY** be defaulted by admission controllers or webhooks
- unspecified fields **MAY** be overridden by e.g. auto-scalers

## Examples

### Annotations/Labels

Annotations and Labels defined by users or other controllers and integrations are not overridden or replaced. Only labels and annotations explicitly set are reconciled to the specified value.

This is important, as Kubernetes operators may use labels to scope their caches.
It also allows humans to add extra labels and annotations for ops or debugging work.

{{< columns >}}
**Desired Object in Package Operator**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec: {}
```
<--->
**Actual Object**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: banana
    something: xxx
  annotations:
    notice: important!
spec: {}
```
{{< /columns >}}

**Result after Reconcile**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
    something: xxx
  annotations:
    notice: important!
spec: {}
```

### Replicas

Fields not explicitly specified, may be defaulted or changed, without being reset by the Package Operator.

{{< columns >}}
**Desired Object in Package Operator**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  # replicas: not set
  template: {}
```
<--->
**Actual Object**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  template: {}
```
{{< /columns >}}

**Result**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  template: {}
```
