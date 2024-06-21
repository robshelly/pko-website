---
title: Multi-Component Packages
draft: false
images: []
weight: 1000
toc: true
---

Multi-component packages are useful when deploying larger and more complex
projects via Package Operator.

They enable users to split the application into multiple components, each one
structured as a standard Package Operator package, and ship them within the
same package image.

Having a single image keeps the build phase simple, while, at the same time,
separating components into their own packages gives the project authors
several advantages:

* component packages may be reused by other applications
* package configuration is isolated from each other
* every package can be rolled back individually

## Structure

*Example structure of a multi-component package*

```text
my-application
├── components
│   ├── backend
│   │   ├── deployment.yaml.gotmpl
│   │   └── manifest.yaml
│   └── frontend
│       ├── deployment.yaml.gotmpl
│       └── manifest.yaml
├── backend-package.yaml.gotmpl
├── frontend-package.yaml.gotmpl
├── manifest.yaml
└── namespace.template.yaml.gotmpl
```

A multi component package starts from the same structure of a simple package:
a set of resources and a `manifest.yaml` file in its root directory. This is
called the **root component**

The `components` folder is the other important piece: it is not part of the
root component, instead **it contains one subfolder for each component**
belonging to the package.

The structure inside each component subfolder matches the one of a standard
Package Operator package: a `manifest.yaml` file with its own resources.

### Important notes

* Only a single level of components is allowed, to limit the complexity of
packages (e.g. `components/backend/components` would not work in the example
above).
* Each component, including the root, has no awareness of the other
components' resources or configuration.

### Enable the feature

Package authors have to opt into using multi-component packages by setting the
`.spec.component` property in the root `PackageManifest`, to prevent breaking
Packages already shipping a `/components` folder. The empty object may be used
in the future to configure the multi-component feature.

```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifest
metadata:
 name: my-application
spec:
 components: {}
```

## Build

The validate and build phase is exactly [the same as a simple package](
/docs/guides/packaging-an-application/#build--validate) (tl;dr `kubectl
package validate` and `kubectl package build`).

It is possible to validate a single component independently running `kubectl
package validate components/<component_folder>`

### Note on the tree command

The `kubectl package tree`, if called from the root directory, will show the
structure of the **root component** only. To see the structure of each
component, a separated call must be made indicating the component subfolder.
Here are some examples considering the structure shown above:

```bash
# show the root component
$ kubectl package tree my-application
my-application
Package app-ns/app
└── Phase deploy-backend
│   ├── package-operator.run/v1alpha1, Kind=Package /app-backend
└── Phase deploy-frontend
    └── package-operator.run/v1alpha1, Kind=Package /app-frontend

# show the backend component
$ kubectl package tree my-application/components/backend
my-application-backend
Package app-ns/app
└── Phase deploy
    └── apps/v1, Kind=Deployment /app

# show the frontend component
$ kubectl package tree my-application/components/frontend
my-application-frontend
Package app-ns/app
└── Phase deploy
    └── apps/v1, Kind=Deployment /app
```

### Note on the update command

The [automatic image digest resolution](docs/guides/image-digests/) feature
can be used in multi-component package **independently in each component**.

Like the tree command described above, the `kubectl package update` command
will consider only the selected component (the root one if called with the
root folder path), read its `manifest.yaml` file and produce a `manifest.lock
yaml` file.

```bash
# this will read my-application/manifest.yaml
# and generate my-application/manifest.lock.yaml
$ kubectl package update my-application

# this will read my-application/components/backend/manifest.yaml
# and generate my-application/components/backend/manifest.lock.yaml
$ kubectl package update my-application/components/backend

# this will read my-application/components/frontend/manifest.yaml
# and generate my-application/components/frontend/manifest.lock.yaml
$ kubectl package update my-application/components/frontend
```

## Deploy

Each component of a multi-component package can be deployed, like any other
simple package, as `Package` or `ClusterPackage` depending on the [scopes](
docs/guides/packaging-an-application/#writing-a-packagemanifest) defined in
its `PackageManifest`.

The `.spec.component` field in `Package` (or `ClusterPackage`) determines
which component will be installed from the specified image.

* If missing or empty, the **root component** will be installed
* If non-empty, its value must match the name of the subfolder containing the
desired component

```yaml
# deploy the root component of my-application
apiVersion: package-operator.run/v1alpha1
kind: Package
metadata:
 name: my-application
spec:
 image: quay.io/my-application
```

```yaml
# deploy the backend component of my-application
apiVersion: package-operator.run/v1alpha1
kind: Package
metadata:
 name: my-application
spec:
 image: quay.io/my-application
 component: backend
```

If the `.spec.component` value doesn't match any of the components bundled in
the image, the package deployment will fail with a related error message.

**IMPORTANT:** since each component is in fact an independent package, each
`Package` or `ClusterPackage` will deploy **only one of them** within a single
definition. Read the *General Tips* section to understand how to setup more
complex deployments.

## General Tips

From a technical perspective, each component, including the root, is
completely unrelated with the others and could contain any kind of resources.

However, if the multi-component package contains a set of parts that are
intended to work together (like `backend` and `frontend` in our example
above), the root component could be used to define the standard layout in
which those components are supposed to be deployed together.

In this way a package user can deploy the single application in the
"quick-way" by deploying the root component and let it take care of wiring the
pieces together, or just one specific part by relying on its component.

This is achieved by placing `Package` or `ClusterPackage` definitions of each
component **inside the root component** (plus any other useful definition,
like the `Namespace` in the example above). This will leverage an already
existing capability of Package Operator.

The root component in the example above, in fact, contains `backend-package
yaml.gotmpl` and `frontend-package.yaml.gotmpl` resources, which are designed
to deploy the two components as independent `Package`s (which will be owned by
the main one) and configure them to work together.

The templating in this case allows to specify a general root component
configuration which will be injected in the configuration of each component
package accordingly.
