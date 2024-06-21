---
title: CEL Conditionals
weight: 1900
images: []
---

This guide covers how objects can be changed or excluded based on
 conditions on the configuration or environment.

Here are a few use cases that can be accomplished using this feature:

- Create different objects depending on the version of Kubernetes.
- Support different installation modes.
- Support multiple deployment topologies.
- Support differences across Kubernetes distributions (e.g. Red Hat OpenShift).

Examples are available in the
[package-operator/examples](https://github.com/package-operator/examples)
repository within the `4_conditionals` folder.

**Common Expression Language - CEL**

The [Common Expression Language (CEL)](https://github.com/google/cel-spec) is
used in Package Operator to declare inclusion rules and other constraints or
conditions.

[Common Expression Language in Kubernetes](https://kubernetes.io/docs/reference/using-api/cel/)

## 1. Go Templates

Go templates can used to exclude objects. \
Package Operator tolerates templates evaluating to empty strings and just
skips these objects.
This also works in multi-document YAML files.

Example:

```yaml
{{ if eq .config.banana "bread" -}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: filter-via-template
  annotations:
    package-operator.run/phase: deploy
{{ end }}
---
# other YAML document
```

## 2. CEL via Annotation

Custom CEL rules can be attached to any object with the `package-operator.run/condition`
annotation. \
The object will only be included if the expression evaluates to true.

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filter-via-annotation
  annotations:
    package-operator.run/phase: deploy
    package-operator.run/condition: .config.banana == "bread"
```

## 3. CEL via Glob

CEL rules can also be attached to folder globs to exclude whole parts of the
file system tree. These rules can be specified as part of the `PackageManifest`.

Example:

```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifest
spec:
  # [...]
  filter:
    paths:
    - expression: '.config.banana == "bread"'
      glob: 'subfolder/**'
```

## 4. Reuse Expressions

To make refactoring and the creation of more complex packages easier,
CEL expressions can be setup once in the `PackageManifest` and
referenced from either Globs or annotations.

Example:

```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifest
spec:
  # [...]
  filter:
    # reusable conditions:
    conditions:
    - name: config_banana_bread
      expression: '.config.banana == "bread"'
    paths:
    - expression: 'cond.config_banana_bread'
      glob: 'subfolder/**'
```

Example use in Annotation:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filter-via-annotation
  annotations:
    package-operator.run/phase: deploy
    # This annotation can contain a CEL expression controlling if this object should be included or not.
    # In this example we are referencing the named condition "config_banana_bread" from the PackageManifest.
    package-operator.run/condition: cond.config_banana_bread
```
