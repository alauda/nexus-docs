---
title: "Configuring Account Access Credentials"
description: ""
weight: 100
---

This document describes the configuration methods for credentials required by Nexus instances.

## Prerequisites

- This document applies to Nexus 3.76.0 and above versions provided by the platform. It is decoupled from the platform using technologies such as Operator.

## Nexus Account Credentials

Create a Secret, select the Opaque type, and add a `password` field in the configuration items:

```yaml
apiVersion: v1
data:
  password: <base64 encode password>
kind: Secret
metadata:
  name: nexus-root-password
type: Opaque
```
