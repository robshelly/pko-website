---
title: Packaging an Application
weight: 100
images: []
---

In this guide, you will deploy a simple nginx web server, using the Package Operator - [Package API](/docs/getting_started/api-reference/#package).

During this guide you will:
* Create a `manifest.yaml` file
* Assign Kubernetes objects to PKO phases
* Take advantage of templates
* Setup test cases for templates
* Build and Validate the Package Operator package

To complete this guide you will need:
* A Kubernetes cluster with Package Operator installed
* The `kubectl-package` CLI plugin
* A container-registry to push images to  
(optional when using tars and kind load)

All files used during this guide are available in the [package-operator/examples](https://github.com/package-operator/examples) repository.

## 1. Start

_Please refer to the files in `/1_applications/1_start` for this step._

When packaging an application for Package Operator, you will need 2 things:
1. One or more Kubernetes manifests (e.g. `deployment.yaml`)
2. A [PackageManifest](/docs/getting_started/api-reference/#packagemanifest) object in a `manifest.yaml` file

### Writing a PackageManifest

Like with any Kubernetes object Group Version Kind contains the version information:
```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifest
```
---

Metadata contains the name of this package:
```yaml
metadata:
  name: nginx
```
---

Packages may be cluster-scope, namespace-scope or both.  
This controls whether you can install this package via `Package` or `ClusterPackage` API.  
Namespaced packages can not contain cluster-scoped objects, like `Namespaces`.
```yaml
spec:
  scopes:
  - Namespaced
```
---
Phases are needed when you need a distinct order in your rollout or teardown.

Examples:
- Ensure an application is completely upgraded before reconfiguring the LB
- Run a database migration before bringing up the new Deployment
- Ensure CRDs are Established before deploying your Operator
```yaml
spec:
  phases:
  - name: deploy
```
---

Probes define how Package Operator interrogates objects under management for status.  
PKO will only continue into the next Phase if all objects passed their availabilityProbe.

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
```

### Assigning objects to phases

Package Operators needs to know in which phase objects belong.  
To assign an object to a phase, simply add an annotation:

```yaml
metadata:
  annotations:
    package-operator.run/phase: deploy
```

### Build & Validate

If you just want to validate the local package contents, use:
```sh
$ kubectl package validate 1_applications/1_start

# example:
Error: Package validation errors:
- Missing package-operator.run/phase Annotation in deployment.yaml#0
```
---
To inspect the parsed hierarchy of your package, use:
```sh
$ kubectl package tree 1_applications/1_start

# example:
nginx
Package namespace/name
└── Phase deploy
    └── apps/v1, Kind=Deployment /nginx-deployment
```
---
And finally to build your package as a container image use:
```sh
# -o will directly output a `podman/docker load` compatible tar.gz of your container image.
# Use this flag if you don't want to push images to a container registry.
$ kubectl package build -t <your-image-url-goes-here> -o nginx.tar.gz 1_applications/1_start
# example: load image into kind nodes:
$ kind load image-archive nginx.tar.gz

# Alternatively push images directly to a registry.
# Assumes you are already logged into a registry via `podman/docker login`
$ kubectl package build -t <your-image-url-goes-here> --push 1_applications/1_start
```

### Deploy

Now that you have build your first Package Operator package, we can deploy it!  
You will find the `Package`-object template in the examples checkout under 1_applications/package.yaml.

```yaml
apiVersion: package-operator.run/v1alpha1
kind: Package
metadata:
  name: my-nginx
spec:
  image: <your-image-url-goes-here>
```

```sh
$ kubectl create -f 1_applications/package.yaml
package.package-operator.run/my-nginx created

# wait for package to be loaded and installed:
$ kubectl get package -w
NAME       STATUS        AGE
my-nginx   Progressing   11s
my-nginx   Available     13s

# success!
$ kubectl get po
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-cd55c47f5-szvh7   1/1     Running   0          79s
```

## 2. Templates

_Please refer to the files in `/1_applications/2_templates` for this step._

Next we want to make sure that the package can be installed multiple times into the same namespace. To accomplish this, we have to make the package more dynamic!

### Go Templates

By renaming `deployment.yaml` into `deployment.yaml.gotmpl`, we can enable [Go template](https://pkg.go.dev/text/template) support. Files suffixed with `.gotmpl` will be processed by the Go template engine before the YAML manifests are parsed.

[TemplateContext](/docs/getting_started/api-reference/#templatecontext) is documented as part of the API.

```
app.kubernetes.io/instance: "{{.Package.Name}}"
```

### Testing Templates

Using a template engine with yaml files can quickly lead to unexpected results.  
To aid with testing, Package Operator includes a simple package testing framework.

Template tests may be configured as part of the `PackageManifest`, by specifying the TemplateContext data to test the template process with.

For each template test, Package Operator will auto-generate fixtures into a `.test-fixtures` folder when running `kubectl package validate` or `build` and compare the output of successive template operations against these fixtures.

```yaml
test:
  template:
  - name: namespace-scope
    context:
      package:
        metadata:
          name: my-cool-name
          namespace: test-ns
```

---

Now make a change to the template and validate the package again:

```yaml
#...
metadata:
  name: "{{.Package.Name}}-deploy"
#...
```

```sh
$ kubectl package validate 1_applications/2_templates
Error: Package validation errors:
- Test "namespace-scope": File mismatch against fixture in deployment.yaml.gotmpl:
  --- FIXTURE/deployment.yaml
  +++ ACTUAL/deployment.yaml
  @@ -1,7 +1,7 @@
   apiVersion: apps/v1
   kind: Deployment
   metadata:
  -  name: "my-cool-name"
  +  name: "my-cool-name-deploy"
     labels:
       app.kubernetes.io/name: nginx
       app.kubernetes.io/instance: "my-cool-name"
```

To continue building the package, either reset your change to the template, edit the fixture or delete the `.test-fixtures` folder and regenerate all fixtures by running the validate command again.

### Upgrade/Deploy

Build your package again using a different tag or image name:

```sh
$ kubectl package build -t <your-image-url-goes-here> --push 1_applications/2_templates
```
---
Edit the `Package` object on the cluster to change the `image:`.
```sh
$ kubectl edit package
```
---
Watch the change being rolled out:
```sh
kubectl get package -w                   
NAME       STATUS      AGE
my-nginx   Available   65m
my-nginx   Progressing   65m
my-nginx   Progressing   65m
my-nginx   Available     65m
```
---
The name of the deployment is now using the name of the Package object, so it's safe to create multiple instances of the same package.  
Just change the name of `1_applications/package.yaml` and deploy your package again:

```
$ kubectl get package
NAME          STATUS      AGE
my-nginx      Available   69m
other-nginx   Available   6s

$ kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
my-nginx      1/1     1            1           6m25s
other-nginx   1/1     1            1           117s
```
