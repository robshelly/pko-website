---
title: Introduction
weight: 100
images: []
---

<div class="package-operator-logo"></div>

Package Operator is a Kubernetes Operator for packaging and managing a collection of arbitrary Kubernetes objects.

Helping users in installing and maintaining complex software on one or multiple clusters.

## Highlights

- No Surprises
  - Ordered Installation and Removal
  - Operating Transparency
- Extensible
  - Declarative APIs
  - Plug and Play
- Cheap Failures and Easy Recovery
  - Rollout History
  - Rollback

## Objectives

**Security, Stability, Transparency, Extensibility**  
(in this order)

### Security

A Kubernetes package manager is entrusted with a very high level of permissions on one or more clusters and also works with secrets as part of package configuration. Without putting security of these credentials first, users will not be able to trust Package Operator.

### Stability

Stability enables any other feature in the Package Operator and makes or breaks its whole value proposition. Because many day-2 operations, like patching, updating and re-configuration can be orchestrated via Package Operator, a misbehaving or broken Package Operator can spell doom to any production environment.

Package Operator commits to stability and extensive automated testing for any feature being implemented.

### Transparency

Stability is never absolute, so it's crucial to be transparent.  
Transparency enables users of the Package Operator to debug and resolve issues, with either their own workloads or the Package Operator itself, in a timely and sane manner.

### Extensibility

The Kubernetes ecosystem is moving _fast_, really _fast_.  
New Operators, APIs, procedures and tools are being created at an astounding pace. 

Package Operator tries to be plug-able, allowing users to use any kind of custom resource registered on the Kubernetes cluster. Facilities of Package Operator are also setup to be overridden, so they can be switched for custom or alternative implementations.
