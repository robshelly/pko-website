---
title: Package Format
draft: false
images: []
weight: 800
toc: true
---

Package Operator packages allow distributing all manifests that make up an application or operator deployment into a single artifact.

This artifact is just an empty container image containing a `PackageManifest` and the Kubernetes manifests in an optionally nested folder structure.

The `Package` or the `ClusterPackage` API is used to load these package container images into the cluster. This loading process will load the image contents into an `ObjectDeployment`.

{{< alert text="Large Packages will automatically use the `ObjectSlice` API to get around etcd object-size limitations.<br>More about this feature can be found on the \"Big Packages\" page." />}}

## Package Structure

| File | Description |
| ---- | ----------- |
| manifest.yaml | `PackageManifest` **required** |
| README.md | Long description or instructions for the package. |
| .png/.svg | Package icon referenced from manifest.yaml. |

## Well-Known Labels

Well-Known Labels are applied automatically to all objects within a Package and the resulting `ObjectDeployment` to provide additional context.

| Label | Description |
| ----- | ----------- |
| `package-operator.run/package` | `Package` object name. |
| `package-operator.run/version` | Version as stated in the `PackageManifest`. |
| `package-operator.run/provider` | Provider as stated in the `PackageManifest`. |

## Well-Known Annotations

Well-Known Annotations are used to control the loading behavior of an object within a package.

| Annotation | Description |
| ---------- | ----------- |
| `package-operator.run/phase` | Assigns the object to a phase when loaded. |

## Example

### PackageManifest - manifest.yaml

```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifest
spec:
  scopes:
  - Cluster
  - Namespaced
  phases:
  - name: phase-1
  - name: phase-2
  - name: phase-3
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
test: {}
```

### Containerfile

```dockerfile
FROM scratch

ADD . /package
```

### Structure

```tree
package
│   manifest.yaml
│   README.md
│   my-icon.png
│   load-balancer.yaml
│
└───frontend
│   │   frontend-deployment.yaml
│   │   frontend-service.yaml
│   │
│   └───cache
│       │   cache-db.yaml
│       │   cache-config.yaml
│       │   ...
│
└───backend
    │   backend-deployment.yaml
    │   backend-config.yaml
    │   ...
```
