---
title: Package Templates
draft: false
images: []
weight: 650
toc: true
---

Package Operator supports templates to pre-process objects with configuration
and environment information before applying them to a Kubernetes Cluster.

Adding the `.gotmpl` suffix to a file, marks it as a
[Go Template](https://pkg.go.dev/text/template). \
All templates have access to
[Template Functions](/docs/api_reference/template-functions/) in addition
to Go templates built-in functionalities. Package Operator also provides
context, configuration and
environment information to the templates:
[Package Operator API - TemplateContext](/docs/api_reference/package-operator-api/#templatecontext).

Templates within packages are only executed **once**,
when a package is being unpacked. \
A package is unpacked when it is being created or
either `.spec.image` or `.spec.config` changes.

## Test Framework

Because running YAML files through a template engine can lead to unexpected
results, Package Operator provides a template test framework to quickly iterate
over changes locally and spot unwanted changes and regressions early.
It can also be used within CI/CD pipelines.

The test framework is configured by adding test cases to
the Package Manifest, like this:

```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifest
# [...]
test:
  template:
    # Name of the test-case
  - name: '15-bananas'
    # Template context to run template engine with.
    context:
      package:
        name: banana
      config:
        replicas: 15
      environment:
        openShift:
          version: v4.12.5
```

Running `kubectl package validate <path-to-package>` will now create a
`.test-fixtures/<test-case>` folder, when it does not already exist.
This folder will contain the output of rendering the package templates
 with the given context information to check if the templates work as expected.

Subsequent runs of the validate command will compare the new output with the
test fixtures and alert on differences. \
Adjust both the templates and the test fixtures at the same time to have the
validation pass.

## Example validate output

```txt
Error: validating package: loading package from files: File mismatch against fixture in deployment.yaml: Testcase "my-testcase"
--- FIXTURE/deployment.yaml
+++ ACTUAL/deployment.yaml
@@ -8,7 +8,7 @@
   annotations:
     package-operator.run/phase: deploy
 spec:
-  replicas: 4
+  replicas: 1
   selector:
     matchLabels:
       app.kubernetes.io/name: nginx
```

## Example Template

Saved as `deployment.yaml.gotmpl`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  # Resolves to the name of the (Cluster)Package object.
  name: "test-stub-{{.package.metadata.name}}"
{{- if eq .package.metadata.namespace ""}}
  # Included if package is installed in a Namespace scope.
  # e.g. via the Package-API and not via the ClusterPackage-API.
  namespace: "{{.package.metadata.name}}"
{{- end}}
  labels:
    app: test-stub
    instance: "{{.package.metadata.name}}"
  annotations:
    # Dumps all available Environment information into this annotation.
    test-environment: {{.environment | toJson | quote}}
    package-operator.run/phase: deploy
spec:
  # Puts in the value supplied via .spec.config.replicas of the Package object.
  replicas: {{.config.replicas}}
  selector:
    matchLabels:
      app: test-stub
      instance: "{{.package.metadata.name}}"
  template:
    metadata:
      labels:
        app: test-stub
        instance: "{{.package.metadata.name}}"
    spec:
      containers:
      - name: test-stub
        # Filled by resolved image digest, if added to PackageManifest.
        # see guide on "Working with image digests"
        image: "{{ index .images "test-stub" }}"
```
