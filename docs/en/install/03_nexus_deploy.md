---
title: "Nexus Instance Deployment"
description: "Nexus Repository is the single source of truth for all your internal and third-party binaries, components, packages, and AI models."
weight: 200
---

This document describes the subscription of Nexus Operator and the functionality for deploying Nexus instances.

## Prerequisites

- This document applies to Nexus 3.76 and above versions provided by the platform. It is decoupled from the platform based on technologies such as Operator.
- Please ensure that Nexus Operator has been deployed (subscribed) in the target cluster, meaning the Nexus Operator is ready to create instances.

## Deployment Planning

Nexus supports various resource configurations to accommodate different customer scenarios. In different scenarios, the required resources and configurations may vary significantly. Therefore, this section describes what aspects need to be considered in deployment planning before deploying Nexus instances, and what the impact of decision points is, to help users make subsequent specific instance deployments based on this information.

### Basic Information

1. The Nexus Operator provided by the platform is based on the community's official Nexus Chart, with enhanced enterprise capabilities such as security vulnerability fixes. It is fully compatible with the community version in terms of functionality, and in terms of user experience, it enhances the convenience of Nexus deployment through optional, customizable templates and other methods.

## Instance Deployment

### Deploying from the `Quick Start Template`

This template is used to quickly create a lightweight Nexus instance, suitable for development and testing scenarios, not recommended for production environments.

- Compute resources: 2 CPU cores, 4 Gi memory
- Storage: Use node local storage, configure the storage node IP and path
- Network access: Use NodePort to access the service, share the node IP with storage, and specify the port

Complete the deployment by filling in the relevant information according to the template prompts.

### Deploying from the `Production Template` Template

This template is used to quickly create a Production Nexus instance, suitable for production scenarios, recommended for production environments.

- Compute resources: 4 CPU cores, 8 Gi memory
- Storage: Use PVC storage, configure the storage class
- Network access: Use Domain to access the service.

Complete the deployment by filling in the relevant information according to the template prompts.

### Deploying from YAML

#### Resource Configuration

Nexus is deployed using StatefulSet, containing 4 containers: 1 business container and 3 logging containers. When configuring resources, focus on the resources used by the business container, while the logging containers can be deployed with default configurations.

```yaml
spec:
  helmValues:
    statefulset:
      container:
        resources:
          requests:
            cpu: 2
            memory: "4Gi"
          limits:
            cpu: 4
            memory: "8Gi"
```

For more information, refer to [Resource description in SonarQube Chart](https://github.com/sonatype/nxrm3-ha-repository/blob/76.0.0/nxrm-ha/values.yaml#L95)

#### Network Configuration

Network configurations are categorized into two types:

- Network configuration based on ingress
- Network configuration based on NodePort

Network configuration based on ingress supports both https and http protocols. An ingress controller needs to be deployed in the cluster in advance.

```yaml
spec:
  helmValues:
    service:
      nexus:
        enabled: true
        type: ClusterIP
    ingress:
      enabled: true
      host: test-ingress-http.example.com
      defaultRule: true
      tls:
        - secretName: "test-tls-cert"
          hosts:
          - test-ingress-https.example.com
```

Network configuration based on NodePort:

```yaml
spec:
  helmValues:
    service:
      nexus:
        enabled: true
        nodePort: 30100
```

#### Storage Configuration

Storage configurations are mainly divided into three categories:

- Storage configuration based on StorageClass
- Storage configuration based on PVC
- Storage configuration based on HostPath

Storage configuration based on StorageClass:

```yaml
spec:
  helmValues:
    storageClass:
      name: <storage-class> ## StorageClass needs to be created in advance
    pvc:
      volumeClaimTemplate:
        enabled: true
      storage: 200Gi   ## Adjust according to actual requirements
```

Storage configuration based on PVC:

```yaml
spec:
  helmValues:
    pvc:
      existingClaim: "nexus-pvc" ## PVC needs to be created in advance
```

Storage configuration based on HostPath:

```yaml
spec:
  helmValues:
    hostPath: /data/nexus ## Select the deployment node and specify the storage path

    statefulset:
      nodeSelector:
        kubernetes.io/hostname: node1 ## Select the deployment node
    nodeSelector:
      kubernetes.io/hostname: node1
```

### Admin Account Configuration

Write the prepared admin password into a secret. The default login username is `admin`.

Create a Secret, select the Opaque type, and add a `password` field in the configuration items:

```yaml
apiVersion: v1
data:
  password: <base64 encode password>
kind: Secret
metadata:
  name: nexus-password
type: Opaque
```

Specify it to Nexus through YAML:

```yaml
spec:
  helmValues:
    secret:
      nexusAdminSecret:
        enabled: true
        existingSecret: "nexus-password"
        secretKey: "password"
```

### Complete YAML Example

NodePort, HostPath, Admin account

```yaml
apiVersion: operator.alaudadevops.io/v1alpha1
kind: Nexus
metadata:
  name: gitlab
  namespace: aesfv-1-testns
spec:
  helmValues:
    secret:
      nexusAdminSecret:
        enabled: true
        existingSecret: aesfv-restore-source
        secretKey: password
      nexusSecret:
        enabled: true
        secretKeyfile: |
          {
            "active": "default",
            "keys": [
              {
                "id": "default",
                "key": "default-key"
              }
            ]
          }
    statefulset:
      container:
        resources:
          requests:
            cpu: 2
            memory: 4Gi
          limits:
            cpu: 4
            memory: 8Gi
        env:
          zeroDowntimeEnabled: true
      nodeSelector:
        kubernetes.io/hostname: node1
    service:
      nexus:
        enabled: true
        nodePort: 30100
    hostPath: /tmp/nexus
```

## Additional Information

### Pod Security Policy

When deploying Nexus in a namespace with Pod Security Policy (PSP) enabled, the following deployment configurations are supported:

- **StorageClass and PVC-based Storage**: Compatible with both `privileged` and `baseline` enforcement modes
- **Node Storage**: Requires `privileged` enforcement mode due to direct host filesystem access requirements

Note: It is recommended to use StorageClass or PVC-based storage in production environments to ensure better security and compliance with Kubernetes security best practices.
