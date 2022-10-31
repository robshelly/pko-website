---
title: Requirements
weight: 300
images: []
---

## Kubernetes Cluster
Package Operator need a Kubernetes cluster to be deployed on. If you don't have a cluster
but still want to play around with Package Operator, there are a few choices of tools to
deploy a Kubernetes cluster locally, such as [minikube](https://kubernetes.io/docs/tasks/tools/#kubectl)
and [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

## kubectl
You will need the Kubernetes CLI, [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl),
to deploy and interact with Package Operator and your packages.

## Container Runtime
You will CLI to build container images, such as [docker](https://docs.docker.com/get-docker/) or
[podman](https://podman.io/getting-started/installation).
