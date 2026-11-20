---
title: "Nexus Instance Deployment"
description: "Nexus Repository is the single source of truth for all your internal and third-party binaries, components, packages, and AI models."
weight: 200
---

This document describes the subscription of Nexus Operator and the functionality for deploying Nexus instances.

::: danger Pod Security Admission Level Restrictions

The Pod Security Admission (PSA) Level requirements for Nexus deployment depend on the storage method used:

1. **HostPath Storage**: The PSA Level **must be configured as `Privileged`**, because `hostPath` volumes are not allowed under the `Restricted` policy.

2. **PVC or StorageClass Storage**: Nexus **can be deployed** under the `Restricted` PSA level.

PSA levels supported by each deployment template:

| Deployment Template  | Supported PSA Levels             |
| -------------------- | -------------------------------- |
| Quick Start Template | Privileged                       |
| Production Template  | Privileged, Baseline, Restricted |

:::

## Prerequisites

- This document applies to Nexus 3.76 and above versions provided by the platform. It is decoupled from the platform based on technologies such as Operator.
- Please ensure that Nexus Operator has been deployed (subscribed) in the target cluster, meaning the Nexus Operator is ready to create instances.

## Deployment Planning

Nexus supports various resource configurations to accommodate different customer scenarios. In different scenarios, the required resources and configurations may vary significantly. Therefore, this section describes what aspects need to be considered in deployment planning before deploying Nexus instances, and what the impact of decision points is, to help users make subsequent specific instance deployments based on this information.

### Basic Information

1. The Nexus Operator provided by the platform is based on the community's official Nexus Chart, with enhanced enterprise capabilities such as security vulnerability fixes. It is fully compatible with the community version in terms of functionality, and in terms of user experience, it enhances the convenience of Nexus deployment through optional, customizable templates and other methods.

### Resource Planning \{#resource-planning}

A Nexus instance runs as a single Pod with one Nexus container and three log containers. Plan the resources of the Nexus container. The log containers need few resources and can keep their default configuration (each container has a limit of 0.2 CPU cores and 512 Mi memory).

For a production instance, set the resource limits of the Nexus container to at least:

- CPU: 4 cores
- Memory: 8 Gi

Sonatype requires at least 8 GB of memory and a JVM heap of at least 2703 MB for Nexus Repository. For details, refer to [Sonatype Nexus Repository System Requirements](https://help.sonatype.com/en/sonatype-nexus-repository-system-requirements.html) and [Nexus Repository Memory Overview](https://help.sonatype.com/en/nexus-repository-memory-overview.html).

#### JVM Memory Settings \{#jvm-memory-settings}

The memory used by the Nexus container consists of the JVM heap, the JVM direct memory, and other JVM memory such as metaspace, thread stacks, and code cache. The total must stay below the memory limit of the container; otherwise, the container is terminated with `OOMKilled`.

By default, the Nexus 3.76 image sets both the heap and the direct memory to 2703 MB (`-Xms2703m -Xmx2703m -XX:MaxDirectMemorySize=2703m`), about 5.3 GB in total. With a memory limit of 8 Gi, this leaves little memory for the rest of the JVM. Set the JVM memory explicitly so that the heap plus the direct memory does not exceed two-thirds of the memory limit, as Sonatype recommends.

For a memory limit of 8 Gi, use a 3 GB heap and 2 GB of direct memory:

```yaml
spec:
  helmValues:
    statefulset:
      container:
        resources:
          limits:
            cpu: 4
            memory: "8Gi"
        env:
          install4jAddVmParams: "-Xms3g -Xmx3g -XX:MaxDirectMemorySize=2g"
```

- Set `-Xms` and `-Xmx` to the same value.
- If you increase the memory limit, increase the heap and the direct memory as well, and keep their sum within two-thirds of the memory limit.

### Database and Supported Scale \{#database-and-supported-scale}

The deployment described in this document uses the embedded H2 database. The H2 database and the file blob stores are both stored in the `/nexus-data` volume of the instance.

According to Sonatype, Nexus Repository with the embedded H2 database supports up to 100,000 components or 200,000 requests per day. Workloads beyond these limits are not supported, and the instance may become unstable. Estimate the number of components and the daily request volume before deployment. For details, refer to [Sonatype Nexus Repository System Requirements](https://help.sonatype.com/en/sonatype-nexus-repository-system-requirements.html) and [Nexus Repository Database](https://help.sonatype.com/en/database-options.html).

## Instance Deployment

### Deploying from the `Quick Start Template`

This template is used to quickly create a lightweight Nexus instance, suitable for development and testing scenarios, not recommended for production environments.

- Compute resources: 2 CPU cores, 4 Gi memory
- Storage: Use node local storage, configure the storage node IP and path
- Network access: Use NodePort to access the service, share the node IP with storage, and specify the port

Complete the deployment by filling in the relevant information according to the template prompts.

### Deploying from the `Production Template` Template

This template is used to quickly create a Production Nexus instance, suitable for production scenarios, recommended for production environments.

- Compute resources: The template presets resource requests and limits for the Nexus container. Before deployment, check the values in the template and make sure that the limits meet the requirements in [Resource Planning](#resource-planning) and that the JVM memory matches the memory limit, as described in [JVM Memory Settings](#jvm-memory-settings).
- Storage: Use PVC storage, configure the storage class
- Network access: Use Domain to access the service.

Complete the deployment by filling in the relevant information according to the template prompts.

### Deploying from YAML

#### Resource Configuration

Nexus is deployed using StatefulSet, containing 4 containers: 1 business container and 3 logging containers. When configuring resources, focus on the resources used by the business container, while the logging containers can be deployed with default configurations. Set the JVM memory according to the memory limit, as described in [JVM Memory Settings](#jvm-memory-settings).

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
        env:
          install4jAddVmParams: "-Xms3g -Xmx3g -XX:MaxDirectMemorySize=2g"
```

For more information, refer to [Resource description in Nexus Chart](https://github.com/sonatype/nxrm3-ha-repository/blob/76.0.0/nxrm-ha/values.yaml#L95)

#### Network Configuration \{#network-configuration}

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

#### Storage Configuration \{#storage-configuration}

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
      storage: 200Gi ## Adjust according to actual requirements
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
          install4jAddVmParams: "-Xms3g -Xmx3g -XX:MaxDirectMemorySize=2g"
      nodeSelector:
        kubernetes.io/hostname: node1
    service:
      nexus:
        enabled: true
        nodePort: 30100
    hostPath: /tmp/nexus
```
