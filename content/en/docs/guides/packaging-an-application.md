---
title: Packaging an Application
weight: 100
images: []
---

In this guide, you will deploy a simple nginx web server, using the Package
Operator - [Package API](/docs/api_reference/package-operator-api/#package).

During this guide you will:

* Create a `manifest.yaml` file
* Assign Kubernetes objects to PKO phases
* Take advantage of templates
* Setup test cases for templates
* Build and Validate the Package Operator package

To complete this guide you will need:

* A Kubernetes cluster with Package Operator installed
* The `kubectl-package` CLI plugin
* Access to a container registry where you can push your images

All files used during this guide are available in the
[package-operator/examples](https://github.com/package-operator/examples) repository.

## 1. Start

_To complete this step, refer to the files in [/1_applications/1_start](https://github.com/package-operator/examples/tree/main/1_applications/1_start)._

When packaging an application for Package Operator, you will need 2 things:

1. One or more Kubernetes manifests (e.g. `deployment.yaml`)
2. A [PackageManifest](/docs/api_reference/package-operator-api/#packagemanifest)
 object in a `manifest.yaml` file

### Writing a PackageManifest

Like with any Kubernetes object, Group Version Kind contains the version information:

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

Packages may be cluster-scoped, namespace-scoped, or both.\
This controls whether you can install this package via `Package` or
`ClusterPackage` API.\
Namespace-scoped packages can not contain cluster-scoped objects, like `Namespaces`.

```yaml
spec:
  scopes:
  - Namespaced
```

---
Phases are needed when you need a distinct order in your rollout or teardown.

Examples:

* Ensure an application is completely upgraded before reconfiguring the load balancer
* Run a database migration before bringing up the new Deployment
* Ensure Custom Resource Definitins (CRDs) are Established before deploying
your Operator

```yaml
spec:
  phases:
  - name: deploy
```

---

Probes define how Package Operator interrogates objects under management for status.\
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

Package Operator needs to know in which phase objects belong.\
To assign an object to a phase, simply add an annotation:

```yaml
metadata:
  annotations:
    package-operator.run/phase: deploy
```

### Build & Validate

If you just want to validate the local package contents, run the
`kubectl package validate` command:

```sh
kubectl package validate 1_applications/1_start
```

\
For example, if you encounter an issue such as a missing annotation,
the output might look like this:

```sh
Error: Package validation errors:
- Missing package-operator.run/phase Annotation in deployment.yaml#0
```

---
To inspect the parsed hierarchy of your package, run the `kubectl package tree` command:

```sh
kubectl package tree 1_applications/1_start
```

\
Example output:

```sh
nginx
Package namespace/name
└── Phase deploy
    └── apps/v1, Kind=Deployment /nginx-deployment
```

---
Finally, build your package as a container image and push the image to a registry.
To directly push images to a registry, assuming you've already
logged in using podman/docker login, use the following command:

```sh
kubectl package build -t <your-image-url-goes-here> --push <path-to-package-contents>
```

\
Example:

```sh
kubectl package build -t quay.io/myquayusername/nginx --push 1_applications/1_start
```

### Deploy

Now that you have built your first Package Operator package, we can deploy it!\
You will find the `Package`-object template in the examples directory under
1_applications/package.yaml. Don't forget to change the package url so it \
corresponds to the one used when building the package.
For example, replace
`image: <your-package-url-goes-here>` with `image: "quay.io/myquayusername/nginx"`.

If the package specifies a configuration set, its values are to be specified in \
a config section within the spec. If a config entry has a default specified in \
the package manifest it may be overridden here. If the package requires values \
that are not defaulted and missing here, the package installation will fail.

Example Package object template:

```yaml
apiVersion: package-operator.run/v1alpha1
kind: Package
metadata:
  name: my-nginx
spec:
  image: <your-package-url-goes-here>
  config:
    nginxImage: "nginx:1.23.3"
```

1. To deploy the package, execute the following command:

   ```sh
   kubectl create -f 1_applications/package.yaml
   ```

   \
   You will receive a confirmation message similar to:

   ```sh
   package.package-operator.run/my-nginx created
   ```

2. Wait for package to be loaded and installed:

   ```sh
   kubectl get package -w
   ```

   \
   The status will change from `Progressing` to `Available`:

   ```sh
   NAME       STATUS        AGE
   my-nginx   Progressing   11s
   my-nginx   Available     13s
   ```

3. Finally, to verify the deployment status, run:

   ```sh
   kubectl get po
   ```

   A "READY" value of `1/1` and a "STATUS" of "Running" indicate a successful deployment.

## 2. Templates

_To complete this step, refer to the files in [/1_applications/2_templates](https://github.com/package-operator/examples/tree/main/1_applications/2_templates)._

Next, we want to make sure that the package can be installed multiple times into
the same namespace. To accomplish this, we have to make the package more dynamic!

### Go Templates

By renaming `deployment.yaml` to `deployment.yaml.gotmpl`, we can enable
[Go template](https://pkg.go.dev/text/template) support. Files suffixed with
`.gotmpl` will be processed by the Go template engine before the YAML manifests
are parsed.

[TemplateContext](/docs/api_reference/package-operator-api/#templatecontext) is \
documented as part of the API. It always contains information like package \
metadata that can be used to reduce reduncancies.

```yaml
app.kubernetes.io/instance: "{{.package.metadata.name}}"
```

Additionally, packages can include a `config` section. This section requires
the config to be specified in the package manifest as an
[OpenAPI specification](https://spec.openapis.org/oas/v3.1.0). It is
recommended to require values that are always needed for package
deployment and set defaults if appropriate.

To inspect the parsed hierarchy of your package when using a `config` section, you
must provide a configuration file with the required values:

Example `config.yaml`:

```yaml
nginxImage: "nginx:1.23.3"
```

Inspect the parsed hierarchy of your package:

```sh
kubectl package tree 1_applications/2_templates/ --config-path config.yaml
```

Alternatively, the config section of a named test template within the manifest \
can be used:

```sh
kubectl package tree 1_applications/2_templates/ --config-testcase namespace-scope
```

### Testing Templates

Using a template engine with yaml files can quickly lead to unexpected results.\
To aid with testing, Package Operator includes a simple package testing framework.

Template tests may be configured as part of the `PackageManifest`, by specifying
the TemplateContext data to test the template process with.

For each template test, Package Operator will auto-generate fixtures into a `.test-fixtures`
folder when running `kubectl package validate` or `build` and compare the output
of successive template operations against these fixtures.

Example of a template test defined in `manifest.yaml`:

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

Make a change to the deployment template:

```yaml
#...
metadata:
  name: "{{.package.metadata.Name}}-deploy"
#...
```

Validate the package again:

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

To continue building the package, either reset your change to the template, edit
the fixture or delete the `.test-fixtures` folder and regenerate all fixtures by
running the validate command again.

### Upgrade/Deploy

This section provides a steps on how to upgrade or
deploy your package using the Package Operator.

1. Build your package again using a different tag or image name:

   ```sh
   kubectl package build -t <your-image-url-goes-here> --push 1_applications/2_templates
   ```

2. Edit the `Package` object on the cluster to change the `image:` to
the new build tag.
  Example:

   ```sh
   kubectl edit package
   ```

   change:

   ```sh
   #...
   spec:
    config:
      nginxImage: nginx:1.23.3
    image: quay.io/myquayusername/nginx:0.0.1
    #...
   ```

   to:

   ```sh
   #...
   spec:
     config:
       nginxImage: nginx:1.23.3
     image: quay.io/myquayusername/nginx:0.0.2
   #...
   ```

3. Monitor the change being rolled out:

   ```sh
   kubectl get package -w
   ```

   The status will transition from `Progressing` to `Available`.

---
The deployment's name now utilizes the name of the Package object, making it safe
to create multiple instances of the same package.

1. Modify the `name` in `1_applications/package.yaml` file and redeploy your package:

   * Change the Package name in the YAML file:

     ```yaml
     #...
     kind: Package
     metadata:
       name: other-nginx
     #...
     ```

   * Redeploy the package:

     ```sh
     kubectl create -f 1_applications/package.yaml
     package.package-operator.run/other-nginx created
     ```

2. Verify the changes:

   * Check the package status:

     ```sh
     $ kubectl get package
     NAME          STATUS      AGE
     my-nginx      Available   69m
     other-nginx   Available   6s
     ```

   * Check the deployments:

     ```sh
     $ kubectl get deploy
     NAME          READY   UP-TO-DATE   AVAILABLE   AGE
     my-nginx      1/1     1            1           6m25s
     other-nginx   1/1     1            1           117s
     ```
