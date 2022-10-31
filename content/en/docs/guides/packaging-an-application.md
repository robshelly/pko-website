---
title: Packaging an Application
weight: 100
images: []
---

## Objectives
* Create a manifest.yaml file
* Add annotations to Kubernetes object manifest files to specify phase
* Package all files in a Containerfile
* Build the image

## Before you begin
This guide assumes you have all the [required software](/docs/getting_started/requirements) installed and have a
Kubernetes cluster with [Package Operator installed](/docs/getting_started/installation).

## Application

The application is a simple nginx webserver deployment. The deployment should also be run in
its own namespace, called nginx. The manifests for these two resources are as follows:

###### namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx
```

###### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

## Package Manifest
The first step in packaging an application is to create a package manifest file. Read more about the package
manifest file on the [Package Format page](/docs/concepts/package-format).
For our application the package manifest file looks like:


###### manifest.yaml
```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifest
metadata:
  name: nginx
spec:
  scopes:
  - Cluster
  phases:
  - name: namespace
  - name: deploy
  availabilityProbes:
  - probes:
    - condition:
        type: Available
        status: "True"
    - fieldsEqual:
        fieldA: .status.updatedReplicas
        fieldB: .status.replicas
    selector:
      kind:
        group: apps
        kind: Deployment
```

A short discussion about the different fields in `.spec`.
### Scopes
Since the package contains a namespace object, which is cluster scoped, the only possible scope for the
package is `Cluster`. You can read more about scopes on the [Scopes page](/docs/concepts/scopes).

### Phases
The namespace must be created before the deployment. Therefore, the Package Manifest file has two phases defined,
`namespace` and `deploy`, in that order. Read more about phases
on the [Phases page](/docs/concepts/phases).


### Availability Probes
This is a standard probe for deployment resources. You can read more about availability probes
on the [Probes page](/docs/concepts/probes).

## Annotations
Now we have to link each object to a phase. This is done by adding a `package-operator.run/phase` annotation to the object.
For example, our `namespace.yaml` file would now look like:

###### namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx
  annotations:
    package-operator.run/phase: namespace
```


## Package Image
Package Operator receives these files via a non-runnable container image. The files have to be contained in the
`/package` directory in the image. Therefore, the container file is very simple:

###### package.Containerfile
```dockerfile
FROM scratch

ADD . /package
```


#### Build the Image
This package image can be built with whichever tool you use for building images, for example:

```shell
podman build -t packageImage -f package.Containerfile .
```

## Next Steps
See the [Installing Packages page](/docs/guides/installing-packages) to see how to deploy packages using Package Operator.
