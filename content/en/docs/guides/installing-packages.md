---
title: Installing Packages
weight: 200
images: []
---

## Objectives
* Create a package object manifest
* Deploy the package object

## Before you begin
This guide assumes there is already an image for the given package. If not, the
[Packaging an Application](/docs/guides/packaging-an-application) or the
[Packaging an Operator](/docs/guides/packaging-an-operator) guide show how to package an application or an operator
respectfully.


## (Cluster)Package Object Manifest
The `(Cluster)Package` API is used to load package container images into a cluster.

The `ClusterPackage` API is used for packages that have `Cluster` in their defined scopes and the
`Package` API is used for packages that have `Namespaced` in their defined scopes.

Read more about scopes on the [Scopes page](/docs/concepts/scopes).

Let's say we want to deploy a package that only has `Cluster` scope. Since basically everything is already contained
in the image, the package object manifest is quite simple.

###### package.yaml
```yaml
apiVersion: package-operator.run/v1alpha1
kind: ClusterPackage
metadata:
  name: example
spec:
  image: packageImage

```

See the
[Package api reference](/docs/getting_started/api-reference#package) and
[ClusterPackage api reference](/docs/getting_started/api-reference#clusterpackage) for
more information.


## Deploy Package Object
The package object manifest can now be deployed using `kubectl`:
```shell
kubectl create -f package.yaml
```
