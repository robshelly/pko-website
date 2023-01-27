---
title: Packaging an Operator
weight: 300
images: []
---

This guide extends on [Packaging an Application](/docs/guides/packaging-an-application/), with the goal to package and deploy an Kubernetes Operator using the Package Operator - [ClusterPackage API](/docs/getting_started/api-reference/#clusterpackage).

During this guide you will:

* Create a `manifest.yaml` file
* Use multiple PKO phases
* Build and Validate the Package Operator package

To complete this guide you will need:

* A Kubernetes cluster with Package Operator installed
* The `kubectl-package` CLI plugin
* A container-registry to push images to\
(optional when using tars and kind load)

All files used during this guide are available in the [package-operator/examples](https://github.com/package-operator/examples) repository.

## 1. Start

_Please refer to the files in `/2_operators/1_start` for this step._

### Writing a PackageManifest

Operators are always installed for the whole cluster, so the package is also scoped for the cluster.

```yaml
spec:
  scopes:
  - Cluster
```

---

Operators require distinct order-of-operations to successfully install.\
The phases list represents these steps.

* `CustomResourceDefinitions` have no pre-requisites, so they come first.
* `Namespaces` also has no dependencies.
* `ServiceAccount` and `RoleBinding` need the `Namespace`
* `Deployment` needs all of the above.

```yaml
spec:
  phases:
  - name: crds
  - name: namespace
  - name: rbac
  - name: deploy
```

Probes define how Package Operator interrogates objects under management for status.\
For this operator we want to add a new probe to ensure the CRDs have been established.

```yaml
spec:
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
  - probes:
    - condition:
        type: Established
        status: "True"
    selector:
      kind:
        group: apiextensions.k8s.io
        kind: CustomResourceDefinition
```

### Assigning objects to phases

Package Operators needs to know in which phase objects belong.\
To assign an object to a phase, simply add an annotation:

```yaml
metadata:
  annotations:
    package-operator.run/phase: crds|namespace|rbac|deploy
```

### Build & Validate

To inspect the parsed hierarchy of your package, use:

```sh
$ kubectl package tree --cluster 2_operators/1_start

# example:
example-operator
ClusterPackage /name
└── Phase crds
│   ├── apiextensions.k8s.io/v1, Kind=CustomResourceDefinition /nginxes.example.thetechnick.ninja
└── Phase namespace
│   ├── /v1, Kind=Namespace /example-operator
└── Phase rbac
│   ├── /v1, Kind=ServiceAccount example-operator/example-operator
│   ├── rbac.authorization.k8s.io/v1, Kind=Role example-operator/example-operator
│   ├── rbac.authorization.k8s.io/v1, Kind=RoleBinding example-operator/example-operator
│   ├── rbac.authorization.k8s.io/v1, Kind=ClusterRole /example-operator
│   ├── rbac.authorization.k8s.io/v1, Kind=ClusterRoleBinding /example-operator
└── Phase deploy
    └── apps/v1, Kind=Deployment example-operator/example-operator
```

---
And finally to build your package as a container image use:

```sh
# -o will directly output a `podman/docker load` compatible tar.gz of your container image.
# Use this flag if you don't want to push images to a container registry.
$ kubectl package build -t <your-image-url-goes-here> -o example-operator.tar.gz 2_operators/1_start
# example: load image into kind nodes:
$ kind load image-archive example-operator.tar.gz

# Alternatively push images directly to a registry.
# Assumes you are already logged into a registry via `podman/docker login`
$ kubectl package build -t <your-image-url-goes-here> --push 2_operators/1_start
```

### Deploy

Now that you have build your first Package Operator package, we can deploy it!\
You will find the `Package`-object template in the examples checkout under 2_operators/clusterpackage.yaml.

```yaml
apiVersion: package-operator.run/v1alpha1
kind: ClusterPackage
metadata:
  name: example-operator
spec:
  image: <your-image-url-goes-here>
```

```sh
$ kubectl create -f 2_operators/clusterpackage.yaml
clusterpackage.package-operator.run/example-operator created

# wait for package to be loaded and installed:
$ kubectl get clusterpackage -w
NAME               STATUS        AGE
example-operator   Progressing   10s
example-operator   Available     22s

# success!
$ kubectl get po -n example-operator
NAME                                READY   STATUS    RESTARTS   AGE
example-operator-5d86f95b4f-z4wfz   1/1     Running   0          17m
```
