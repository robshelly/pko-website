---
title: Installation
weight: 400
images: []
---

Package Operator can be installed in multiple different ways.
Check the latest available release at [Package Operator Releases](https://github.com/package-operator/package-operator/releases).

## Via Package Operator
Package Operator is able to bootstrap and upgrade itself using a special self-bootstrap-job.

Make sure `KUBECONFIG` is defined and the config points at your Kubernetes cluster.
Then you can deploy Package Operator to bootstrap itself:

```
kubectl create -f https://github.com/package-operator/package-operator/releases/download/latest/self-bootstrap-job.yaml
```
This will not install the webhook server.


## Via Mage
Clone the [Package Operator](https://github.com/package-operator/package-operator) repository.
Make sure `KUBECONFIG` is defined and the config points at your Kubernetes cluster. From the root of the repository run
`VERSION="<Release to install>" ./mage deploy`.

This will install the Package Operator Manager and the webhook server.

## Via Manifests
Package Operator has a single yaml file, `install.yaml`, which includes the manifests of all resources that make up
Package Operator. Therefore, Package Operator can be installed with the single command:
```
https://raw.githubusercontent.com/package-operator/package-operator/main/install.yaml
```
This will not install the webhook server.
