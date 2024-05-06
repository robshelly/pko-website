---
title: Collision Prevention
weight: 350
images: []
mermaid: true
---

When updating objects on a Kubernetes cluster, multiple actors may start
to update the same object. In the worst case, this may cause kube-apiserver
unavailability as multiple controllers or other automation produce high load
on both the apiserver and etcd.

To prevent this situation, Package Operator is by default refusing to operate
on objects that have not been created by Package Operator itself.

## Disable Collision Protection

Package Operator offers 3 different protection levels that can be set on
the object-level. This allows to disable collision protection when selected
objects are already present and should be "adopted" by Package Operator.

- **Prevent**\
  The default setting.\
  Prevents any operation on objects not created by PKO.
- **IfNoController**\
  Allows operations on objects without an owner reference.
  This includes objects created directly by users,
  but excludes objects created via controllers.
- **None**\
  Disables all collision prevention checks.\
  *Only use this settings when you know what you are doing!*

### Package

To change the collision protection settings for objects within a package,
attach the: `package-operator.run/collision-protection` annotation to the
objects manifest YAML file.

### ObjectDeployment/-Set

When using the ObjectDeployment or ObjectSet APIs directly, the collision
protection setting can be set next to each object:
[API Reference - ObjectSetObject](/docs/getting_started/api-reference/#objectsetobject)
