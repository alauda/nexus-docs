# Nexus Migration Guide: 3.69.0 to 3.76.0 (Alauda Build of Nexus Operator Version v3.76.z)

## Overview

This document provides detailed instructions for migrating Nexus Repository Manager from version 3.69.0 to 3.76.0. Due to Nexus discontinuing OrientDB support after version 3.70, a manual data migration process is required.

## Migration Process

### 1. Data Backup

Create backup directory:

```bash
mkdir -p nexus-backup
```

Set environment variables:

```bash
export INSTANCE_NAME=<nexus instance name> 
export INSTANCE_NAMESPACE=<nexus instance namespace>
export POD_NAME=$(kubectl -n ${INSTANCE_NAMESPACE} get pod -l release=${INSTANCE_NAME} | grep ${INSTANCE_NAME} | awk '{print $1}')
```

#### 1.1 Service Shutdown

```bash
kubectl -n ${INSTANCE_NAMESPACE} delete service -l release=${INSTANCE_NAME}
```

#### 1.2 Backup Blob Storage

```bash
mkdir -p nexus-backup/blobs
kubectl -n ${INSTANCE_NAMESPACE} cp ${POD_NAME}:/nexus-data/blobs nexus-backup/blobs
```

#### 1.3 Database Backup

1. Navigate to: Admin Dashboard > System > Tasks
2. Create new task:
   - Type: Admin - Export databases for backup
   - Task name: [Your choice]
   - Backup location: [Specify path]
   - Frequency: Manual
3. Execute task and verify completion status

### 2. Database Migration

#### 2.1 OrientDB to H2 Conversion

Set environment variables:

```bash
export INSTANCE_NAME=<nexus instance name> 
export INSTANCE_NAMESPACE=<nexus instance namespace>
export POD_NAME=$(kubectl -n ${INSTANCE_NAMESPACE} get pod -l release=${INSTANCE_NAME} | grep ${INSTANCE_NAME} | awk '{print $1}')
```

Access container and download migration tool:

```bash
kubectl -n ${INSTANCE_NAMESPACE} exec -it ${POD_NAME} -- bash
cd /nexus-data/backup
wget https://download.sonatype.com/nexus/nxrm3-migrator/nexus-db-migrator-3.70.3-01.jar -O nexus-db-migrator.jar
```

Alternative download method:

```bash
kubectl -n ${INSTANCE_NAMESPACE} cp nexus-db-migrator.jar ${POD_NAME}:/nexus-data/backup
```

Execute migration:

```bash
# The nexus.mv.db file will be generated in the backup directory, indicating migration completion
java -Xmx16G -Xms16G -XX:+UseG1GC -XX:MaxDirectMemorySize=28672M -jar nexus-db-migrator.jar --migration_type=h2
```


Backup converted database:

```bash
kubectl -n ${INSTANCE_NAMESPACE} cp ${POD_NAME}:/nexus-data/backup/nexus.mv.db nexus-backup/nexus.mv.db
```

### 3. New Instance Deployment

Since data and storage are imported from the original instance, when deploying the new instance, you only need to maintain the same access method as the original instance.

#### 3.1 Configuration Options

For NodePort access:

```yaml
helmValues:
   service:
     nexus:
       enabled: true
       nodePort: <port>
```

For HTTP ingress:

```yaml
service:
  nexus:
    enabled: true
    type: ClusterIP

ingress:
  enabled: true
  host: <domain>
  defaultRule: true
```

For HTTPS ingress:

```yaml
helmValues:
   service:
     nexus:
       enabled: true
       type: ClusterIP
   
   ingress:
     enabled: true
     host: <domain>
     defaultRule: true
     tls:
       - secretName: <tls-secret>
         hosts:
         - <domain>
```

#### 3.2 Data Import

Set new instance variables:

```bash
export INSTANCE_NAME=<new nexus instance name> 
export INSTANCE_NAMESPACE=<new nexus instance namespace>
export POD_NAME=$(kubectl -n ${INSTANCE_NAMESPACE} get pod -l app.kubernetes.io/instance=${INSTANCE_NAME} | grep ${INSTANCE_NAME} | awk '{print $1}')
```

Clear existing data:

```bash
kubectl -n ${INSTANCE_NAMESPACE} exec -it ${POD_NAME} -- rm -rf /nexus-data/blobs /nexus-data/db/nexus.mv.db 
```

Import backup data:

```bash
kubectl -n ${INSTANCE_NAMESPACE} cp nexus-backup/blobs ${POD_NAME}:/nexus-data
kubectl -n ${INSTANCE_NAMESPACE} cp nexus-backup/nexus.mv.db ${POD_NAME}:/nexus-data/db
```

Then restart the Nexus Pod.

### 4. Verification Steps

1. Verify pod status:

    ```bash
    kubectl -n ${INSTANCE_NAMESPACE} get pods -l app.kubernetes.io/instance=${INSTANCE_NAME}
    ```

2. Validation checklist:
   - [ ] Web UI accessibility
   - [ ] Repository visibility
   - [ ] Artifact upload functionality
   - [ ] Artifact download functionality
   - [ ] User authentication
   - [ ] Repository permissions

3. Post-migration cleanup:
   - Remove old instance after successful verification
   - Archive backups according to retention policy
