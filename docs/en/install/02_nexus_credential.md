---
title: "Configuring Account Access Credentials"
description: ""
weight: 100
---

::: danger Deprecated: Nexus 3.81 (nexus-ce-operator v3.81.1 and v3.81.2)

Due to restrictive limitations in Nexus 3.81 Community Edition, **this version has been discontinued**. Once usage reaches the quota limits (100,000 total components and 200,000 requests per day), the instance will cease to function properly.

**If you have already upgraded to version 3.81, please follow the [Rollback to Nexus 3.76](../howto/04_rollback_to_3.76.mdx) guide to downgrade.**

For more information about Community Edition limitations, see [Community Edition Limitations](https://help.sonatype.com/en/ce-onboarding.html#community-edition-limitations).

:::

This document describes the configuration methods for credentials required by Nexus instances.

## Prerequisites

- This document applies to Nexus 3.76.0 and above versions provided by the platform. It is decoupled from the platform using technologies such as Operator.

## Nexus Account Credentials \{#nexus-credentials}

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
