---
title: Scopes
draft: false
images: []
weight: 600
toc: true
mermaid: true
---

The two possible values for `scopes` are `Cluster` and `Namespaced`.

`Cluster` scope allows the package to deploy cluster scoped object, such as CRDs.
It also allows for deploying namespaced resources in multiple namespaces.

`Namespaced` scope restricts the package to only installing namespaced resources,
and only into the namespace that the package is in.

Just one or both scopes can be specified for a single package.

The `scopes` given in the Package Manifest file determine which package resource
can be used to deploy the package. If only `Cluster` or only `Namespaced` is given
as a scope, then the package must be deployed as a
[ClusterPackage](/docs/getting_started/api-reference#clusterpackage)
or [Package](/docs/getting_started/api-reference#package) respectfully.
If both scopes are given, then the package can be deployed as either resource.
