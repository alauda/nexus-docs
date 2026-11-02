# Nexus Migration Guide: 3.69.0 to 3.76.0 (Alauda Build of Nexus Operator Version v3.76.z)

## Overview

This document provides detailed instructions for migrating Nexus Repository Manager from version 3.69.0 to 3.76.0. Due to Nexus discontinuing OrientDB support after version 3.70, a manual data migration process is required.

## Prerequisites

- [Install Alauda Build of Nexus Operator v3.76.z version](../install/01_installation_guide.mdx)
- Ensure sufficient system resources are available for the upgrade
- Use product functionality to upgrade Nexus to version 3.69.0

## Migration Process

### 1. Data Backup

Create backup directory (ensure sufficient local disk space for backup):

```bash
mkdir -p nexus-backup
```

Set environment variables:

```bash
export INSTANCE_NAME=<nexus instance name> 
export INSTANCE_NAMESPACE=<nexus instance namespace>
export POD_NAME=$(kubectl -n ${INSTANCE_NAMESPACE} get pod -l release=${INSTANCE_NAME} | grep ${INSTANCE_NAME} | awk '{print $1}')
```

#### 1.1 Create Database Backup Task

Create the database export task in the Nexus Web UI **before** shutting down the service, because the Web UI is no longer reachable after the Service is deleted. The task is only created here; it is executed through the REST API in step 1.3.

1. Log in to the Nexus Web UI as an administrator and navigate to: Administration (gear icon) > System > Tasks. If a setup wizard is shown on the first sign-in, complete it first (choose an anonymous access option to continue).
2. Create new task:
   - Type: Admin - Export databases for backup
   - Task name: [Your choice]
   - Backup location: `/nexus-data/migration-backup` - The directory must be on the Nexus data volume, which is persistent. Do not use `/nexus-data/backup`: instances created by the old version operator mount a separate, non-persistent volume there by default, and its content is lost when the Pod is recreated. The database migration in step 2 reads the exported files from this directory.
   - Task enabled: checked (a disabled task cannot be run through the REST API and returns `405`)
   - Task frequency: Manual
3. Save the task. Do not run it yet.
4. Disable the scheduled tasks that modify data (for example cleanup policies and compact blob store tasks), so that nothing changes while the backup is running.

#### 1.2 Service Shutdown

Delete the Nexus Service to stop client traffic that reaches Nexus through the Service (NodePort and Ingress). This does not delete the Pod or its data, and it does not stop the scheduled tasks inside Nexus, which is why they are disabled in step 1.1.

First uninstall the old version operator, otherwise the Service is automatically recreated after deletion. The Subscription of the old version operator is named `nexus-operator`. Do not delete the `nexus-ce-operator` Subscription: it belongs to the new version operator installed in the prerequisites.

```bash
kubectl get subscription --all-namespaces | grep -w nexus-operator
kubectl -n <namespace shown in the first column> delete subscription.operators.coreos.com nexus-operator
```

The running Nexus Pod and its data are not affected by uninstalling the operator.

Then check that only the Services of the old instance are listed, and delete them:

```bash
kubectl -n ${INSTANCE_NAMESPACE} get service -l release=${INSTANCE_NAME}
kubectl -n ${INSTANCE_NAMESPACE} delete service -l release=${INSTANCE_NAME}
```

#### 1.3 Database Backup

Run the task created in 1.1 through the Nexus REST API from inside the Pod. The API is called on `localhost:8081`, so it does not depend on the deleted Service.

Set the administrator credentials. Use the password you set for the `admin` user: the value of the admin password Secret used at install time, or the password you changed it to in the Web UI. On an instance whose initial password was never changed, it is stored in `/nexus-data/admin.password` inside the Pod.

```bash
export NEXUS_ADMIN_USER=admin
export NEXUS_ADMIN_PASSWORD='<admin-password>'
```

Get the ID of the database export task (task type `db.backup`). If the query returns no items, list all tasks with `localhost:8081/service/rest/v1/tasks` and pick the one named after the task you created in step 1.1. If a context path is configured for Nexus, add it in front of `/service/rest/...` in all commands below.

```bash
kubectl -n ${INSTANCE_NAMESPACE} exec ${POD_NAME} -- curl -s -u ${NEXUS_ADMIN_USER}:${NEXUS_ADMIN_PASSWORD} 'localhost:8081/service/rest/v1/tasks?type=db.backup'

# output:
# {
#   "items" : [ {
#     "id" : "b379f8cb-ea52-4bae-9eed-84aed7686a63",
#     "name" : "nexus-db-export",
#     "type" : "db.backup",
#     "message" : null,
#     "currentState" : "WAITING",
#     "lastRunResult" : null,
#     "nextRun" : null,
#     "lastRun" : null
#   } ],
#   "continuationToken" : null
# }
```

Run the task:

```bash
export TASK_ID=<task-id>  # replace with the id returned above
kubectl -n ${INSTANCE_NAMESPACE} exec ${POD_NAME} -- curl -s -o /dev/null -w "%{http_code}\n" -u ${NEXUS_ADMIN_USER}:${NEXUS_ADMIN_PASSWORD} -X POST localhost:8081/service/rest/v1/tasks/${TASK_ID}/run

# Code  Description
# 204   Task was run
# 404   Task not found
# 405   Task is disabled
```

Check the task status. Repeat until `currentState` is `WAITING`, `lastRunResult` is `OK` and `lastRun` shows the time you started the run; the export may take a while for large databases:

```bash
kubectl -n ${INSTANCE_NAMESPACE} exec ${POD_NAME} -- curl -s -u ${NEXUS_ADMIN_USER}:${NEXUS_ADMIN_PASSWORD} localhost:8081/service/rest/v1/tasks/${TASK_ID}

# output (lastRun will vary):
# {
#   "id" : "b379f8cb-ea52-4bae-9eed-84aed7686a63",
#   "name" : "nexus-db-export",
#   "type" : "db.backup",
#   "message" : "Admin - Export databases for backup",
#   "currentState" : "WAITING",
#   "lastRunResult" : "OK",
#   "nextRun" : null,
#   "lastRun" : "2025-11-03T15:21:50.302+00:00"
# }
```

Verify that the exported `.bak` files exist in the backup location:

```bash
kubectl -n ${INSTANCE_NAMESPACE} exec ${POD_NAME} -- ls /nexus-data/migration-backup

# output (timestamps will vary):
# analytics-2025-11-03-15-21-50-3.69.0-02.bak
# component-2025-11-03-15-21-50-3.69.0-02.bak
# config-2025-11-03-15-21-50-3.69.0-02.bak
# security-2025-11-03-15-21-50-3.69.0-02.bak
```

If `lastRunResult` is `FAILED`, check the Pod logs for the task ID:

```bash
kubectl -n ${INSTANCE_NAMESPACE} logs ${POD_NAME} | grep ${TASK_ID} -A 10 -B 10
```

#### 1.4 Backup Blob Storage

Back up the blob storage after the database export, so that every blob referenced by the exported database is included in the backup.

```bash
mkdir -p nexus-backup/blobs
kubectl -n ${INSTANCE_NAMESPACE} cp ${POD_NAME}:/nexus-data/blobs nexus-backup/blobs
```

### 2. Database Migration

#### 2.1 OrientDB to H2 Conversion

Set environment variables:

```bash
export INSTANCE_NAME=<old nexus instance name> 
export INSTANCE_NAMESPACE=<old nexus instance namespace>
export POD_NAME=$(kubectl -n ${INSTANCE_NAMESPACE} get pod -l release=${INSTANCE_NAME} | grep ${INSTANCE_NAME} | awk '{print $1}')
```

**Note:** The migration tool runs inside the Nexus container, next to the Nexus process, so the Java options below count against the memory limit of that container, and the container is killed with `OOMKilled` if the limit is exceeded. Increase the memory of the Nexus container before running the tool, or lower the Java options. The operator was uninstalled in step 1.2, so edit the workload of the instance directly.

Access container and download migration tool:

```bash
kubectl -n ${INSTANCE_NAMESPACE} exec -it ${POD_NAME} -- bash
cd /nexus-data/migration-backup
wget https://download.sonatype.com/nexus/nxrm3-migrator/nexus-db-migrator-3.70.3-01.jar -O nexus-db-migrator.jar
```

The `cd` and `wget` commands above run inside the container shell, and so does the migration command below.

The Pod usually has no Internet access. If direct network download is not available, you can download the file locally first and then copy it to the container:

```bash
kubectl -n ${INSTANCE_NAMESPACE} cp nexus-db-migrator.jar ${POD_NAME}:/nexus-data/migration-backup
```

Execute migration (inside the container shell opened above):

```bash
# The nexus.mv.db file will be generated in the backup directory, indicating migration completion
java -Xmx16G -Xms16G -XX:+UseG1GC -XX:MaxDirectMemorySize=28672M -jar nexus-db-migrator.jar --migration_type=h2
```

The tool asks `Do you want to continue [y/n]?`; answer `y`. Run it in an interactive shell (`exec -it`, as above), otherwise it fails with `NoSuchElementException: No line found` when it tries to read the answer. Without a terminal, pipe the answer in instead, for example `echo y | kubectl -n ${INSTANCE_NAMESPACE} exec -i ${POD_NAME} -- sh -c 'cd /nexus-data/migration-backup && java <same Java options as above> -jar nexus-db-migrator.jar --migration_type=h2'` (use `-i` without `-t`). If you run the tool again, first remove the files of the previous run in the backup directory (`rm -rf nexus.mv.db component config security`), otherwise it stops with `Existing database files found`. The Java options above assume a container memory limit well above 16G; for smaller databases and limits, lower `-Xmx`, `-Xms` and `-XX:MaxDirectMemorySize` accordingly.

Type `exit` to leave the container shell. The following command runs on your workstation.

Backup converted database:

```bash
kubectl -n ${INSTANCE_NAMESPACE} cp ${POD_NAME}:/nexus-data/migration-backup/nexus.mv.db nexus-backup/nexus.mv.db
```

### 3. New Instance Deployment

Since data and storage are imported from the original instance, when deploying the new instance, you only need to maintain the same access method as the original instance.

**Note:** Using the same NodePort and Ingress configuration as the old version will cause conflicts and prevent creation. Configure new values instead, and after successful migration, delete the old version instance and then modify the external exposure settings.

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

Clear existing data. The new instance creates an empty database and blob directory on its first start, so this step is required even when the data volume is new:

```bash
kubectl -n ${INSTANCE_NAMESPACE} exec -it ${POD_NAME} -- rm -rf /nexus-data/blobs /nexus-data/db/nexus.mv.db 
```

Import backup data:

```bash
kubectl -n ${INSTANCE_NAMESPACE} cp nexus-backup/blobs ${POD_NAME}:/nexus-data
kubectl -n ${INSTANCE_NAMESPACE} cp nexus-backup/nexus.mv.db ${POD_NAME}:/nexus-data/db
```

Then restart the Nexus Pod.

**Note:** After the import, the `admin` password of the new instance is the password of the old instance. The password from the new instance's admin password Secret is no longer valid.

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
   - Remove old instance after successful verification (requires reinstalling the old version operator to complete old instance resource cleanup)
   - Archive backups according to retention policy
