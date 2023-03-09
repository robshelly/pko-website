---
title: Working with image digests
weight: 400
images: []
---

This guide explains how to leverage the **automatic image digest resolution**
feature when creating a `Package`, which allows the creator to easily replace
the image tags with their respective digests inside the packaged resources.

This enables a stricter control over which images will be included in the
package, since the digest is the ultimate way to uniquely identify an image,
while the tag can identify different ones over time (think of `latest` that gets
updated every time).

All files used during this guide are available in the
[package-operator/examples](https://github.com/package-operator/examples)
repository.

## 1. Start

_Please refer to the files in `/3_image_digests/1_start` for this step. It is
based on [step 1](/docs/guides/packaging-an-application/#1-start) of the
application packaging guide with the addition of some
[templating](/docs/guides/packaging-an-application/#go-templates).
Make sure you understand these concepts before continuing.

### Add images to PackageManifest

In the `.spec.images` field of the `manifest.yaml` file, specify the list of
images needed, each one associated with its own custom label:

```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifest
metadata:
  name: nginx
spec:
  ...
  images:
  - name: webserver
    image: nginx:1.23.3
  - name: base
    image: registry.access.redhat.com/ubi9/ubi-minimal:9.1
```

### Resolve digests

The Package Operator CLI supports a new subcommand: `update`, which resolves the
image tags in the manifests to their corresponding digests. This information is
then stored in a new file called `manifest.lock.yaml` (a.k.a. _lock file_).

The lock file is **mandatory** if the manifest spec contains some images,
otherwise the build will fail.

To generate or update the lock file, issue the following command:

```bash
kubectl package update .  # this assumes we're in the package root folder
```

Here is an example of the output (digests are the ones at the time of writing):

```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifestLock
metadata:
  creationTimestamp: "2023-03-09T13:45:10Z"
spec:
  images:
  - digest: sha256:aa0afebbb3cfa473099a62c4b32e9b3fb73ed23f2a75a65ce1d4b4f55a5c2ef2
    image: nginx:1.23.3
    name: webserver
  - digest: sha256:61925d31338b7b41bfd5b6b8cf45eaf80753d415b0269fc03613c5c5049b879e
    image: registry.access.redhat.com/ubi9/ubi-minimal:9.1
    name: base
```

**IMPORTANT:** if a new image is pushed to its registry, causing an already
specified tag to point to a different digest, the value in the lock file won't
be updated until the `update` subcommand is **explicitly reissued**.

### Use images map in the templates

To use these values in the real package resources, the templating context is
enriched with a map named `images`, in which each image is identified by a key,
consisting of the custom label, pointing to the full image reference that uses
the resolved digest.

Since this is a standard map, it can be used anywhere in any type of resource
with the standard templating syntax.

Here we have two examples. The first one is a `Deployment` that uses the _dot_
syntax to access the map:

```gotemplate
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
  name: nginx-deployment
  ...
spec:
  ...
  template:
    ...
    spec:
      containers:
        - name: nginx
          image: "{{.images.webserver}}"
          ...
```

and the second is a `ConfigMap` that uses the _index_ syntax instead:

```gotemplate
apiVersion: v1
kind: ConfigMap
metadata:
  ...
  name: example-configmap
  ...
data:
  image_with_digest: "{{index .images "base"}}"
```

### Resolved values in deployed package

Here is an examples of the same two resources once deployed in a cluster:

The `Deployment` first:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
  name: nginx-deployment
  ...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: nginx
        image: docker.io/library/nginx@sha256:aa0afebbb3cfa473099a62c4b32e9b3fb73ed23f2a75a65ce1d4b4f55a5c2ef2
        ...
```

and finally the `ConfigMap`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
  name: example-configmap
  ...
data:
  image_with_digest: registry.access.redhat.com/ubi9/ubi-minimal@sha256:61925d31338b7b41bfd5b6b8cf45eaf80753d415b0269fc03613c5c5049b879e
```
