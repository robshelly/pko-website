---
title: Object Templates
draft: false
images: []
weight: 600
toc: true
mermaid: true
---

[ClusterObjectTemplate](/docs/getting_started/api-reference#clusterobjecttemplate)
and [ObjectTemplate](/docs/getting_started/api-reference#objecttemplate) are APIs
defined in Package Operator. These APIs make it possible to create objects by
templating a manifest and injecting values retrieved from other arbitrary source
objects. The source objects are then continuously monitored and any change in the
source values result in an updated templated object.

A subset of this functionality can be achieved by mounting secrets or
configmaps, however there are multiple benefits to using the ObjectTemplate
API. First, values can be sourced from any arbitrary kubernetes object, not
just secrets or configmaps. Second, when a source value is updated a new
version of the templated object will be created i.e. the value will not just
be silently updated. Third, when a source value is specified as optional and
is not present, package operator will continue to reconcile the templated object.
At some point, if the source value becomes available, package operator will
eventually pick up the value and recreate the templated object.

## Example

Say we have an `ObjectTemplate` called `example-object-template`
which templates a package called `test-stub`.

```yaml
apiVersion: package-operator.run/v1alpha1
kind: ObjectTemplate
metadata:
  name: example-object-template
  namespace: default
spec:
  sources:
  - apiVersion: v1
    items:
    - destination: .nice_to_have_metadata
      key: .data.nice_to_have_metadata
    kind: ConfigMap
    name: metadata-config
    namespace: default
    optional: true
  - apiVersion: v1
    items:
      - destination: .database
        key: .data.database
    kind: ConfigMap
    name: database-config
    namespace: default
    optional: false
  template: |
    apiVersion: package-operator.run/v1alpha1
    kind: Package
    metadata:
      name: test-stub
    spec:
      image: "quay.io/package-operator/test-stub-package:v1.0.0-47-g3405dde"
      config: {{toJson .config}}
```

**IMPORTANT:** the `key` and `destination` values must be JSONPaths, i.e. have a
leading dot.

`example-object-template` sources values from `database-config`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
  namespace: default
data:
  database: database-123
```

and `metadata-config`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metadata-config
  namespace: default
data:
  nice_to_have_metadata: useful-label
```

The `database-config` source is not optional, so we have to make sure to create it
before creating `example-object-template`:

```shell
kubectl create -f database-config.yaml
```

We can now create `example-object-template`:

```shell
kubectl create -f example-object-template.yaml
```

We can retrieve the yaml of the created `package` with:

```shell
kubectl get package test-stub -o yaml
```

which will return something like:

```yaml
apiVersion: package-operator.run/v1alpha1
kind: Package
metadata:
  creationTimestamp: "2023-03-17T14:09:05Z"
  finalizers:
  - package-operator.run/loader-job
  generation: 1
  labels:
    package-operator.run/cache: "True"
  name: test-stub
  namespace: default
  ownerReferences:
  - apiVersion: package-operator.run/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: ObjectTemplate
    name: example-object-template
    uid: f8072c83-7760-4baf-9155-25901c7999eb
  resourceVersion: "201176"
  uid: 4ceda067-14b7-439d-8040-e983dd7d1545
spec:
  config:
    "":
      database: database-123
  image: quay.io/package-operator/test-stub-package:v1.0.0-47-g3405dde
status:
  conditions:
  - lastTransitionTime: "2023-03-17T14:09:05Z"
    message: Unpack job in progress
    observedGeneration: 1
    reason: Unpacking
    status: "False"
    type: Unpacked
  phase: NotReady
```

We can see that the `database` value from `database-config` was
successfully injected into the package's config field. Because the
`metadata-config` configmap does not exist and the item is optional,
the creation of the package went through normally, and we just don't have
a `nice_to_have_metadata` entry in the config.

If we run `kubectl describe objecttemplate example-object-template`, we can see
that the status conditions from `test-stub` where copied over to
`example-object-template`.

Now if we create the `metadata-config` configmap

```shell
kubectl create -f metadata-config.yaml
```

and retrieve the package yaml again (`kubectl get package test-stub -o yaml`),
we see that the `generation` has been increased and that `nice_to_have_metadata`
has been injected into the package's `config` field.
