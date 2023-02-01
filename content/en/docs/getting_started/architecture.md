---
title: Architecture
weight: 200
images: []
---

## Package Operator Manager

The main work of Package Operator is done by the Package Operator Manager. This
component is run as a deployment with a single replica. It contains the controllers
for `(Cluster)Package`, `(Cluster)ObjectDeployment`, and `(Cluster)ObjectSet` resources.
It also contains a controller for `(Cluster)ObjectSetPhase` objects. This controller
reconciles `(Cluster)ObjectSetPhase`s with the `.spec.class` set to `default`.

Package Operator Manager also contains functionality to copy its own binary and
load packages, which is discussed in the [Loading Package Images Section](#loading-package-images),
and functionality to allow Package Operator to bootstrap itself, which is discussed
on the [Installation Page](/docs/getting_started/installation#via_package_operator).

## Webhooks

There is a validation webhook server that can optionally be run with package operator.
This webhook server verifies the immutability of certain fields of `ObjectSet`
and `ObjectSetPhase` resources. If one of these immutable fields is changed in
an update, the webhook server will disallow the update.

## Loading Package Images

A package is a single artifact that contains all manifests, configuration,
and metadata needed to run an application or operator. This single artifact takes
the form of a non-runnable container image. This image is supplied to Package
Operator in a `(Cluster)Package` object.

When Package Operator reconciles a `(Cluster)Package` object, it needs to access
the content of the package image. Package Operator does this by deploying a separate
Kubernetes Job. This Job has two containers, an init container called `prepare-loader`
and a regular container called `package-loader`.

The `prepare-loader` container uses the `package-operator-manager` image and copies
the `package-operator-manager` binary to a shared volume that both containers in
the Job have access to. We have to do this because as mentioned before, the package
image is non-runnable, so we have to assume it has no shell or copy utilities to
unpack the files it contains.

The `package-loader` container then uses the `package-operator-manager` binary to
create an `ObjectDeployment` that contains all the necessary information from the
package image. The Job then finishes and is garbage collected.
