---
id: ee683c51-8ba4-410a-9263-7b787780b2d5
binder: binder-mvvle32a-1
folder: folder-mvw0u6xp
---
DevOps | devplus-platform/DATABASE_OPERATIONS.md

Source: DevPlusInc/devplus-platform/DATABASE_OPERATIONS.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, DATABASE_OPERATIONS.md).

# Database Operations Guide

This guide provides common MongoDB database operations using kubectl for the DevPlus application.

It targets the in-cluster MongoDB that Tilt deploys in `local` mode: a single-member
replica set (`rs0`) in the StatefulSet `mongodb` (pod `mongodb-0`) in the tenant
namespace, with one database per tenant named after the tenant. In `atlas` and `aks`
modes there is no in-cluster MongoDB: the tenant database lives on the Atlas cluster
`devplus-dev.wnzhnl.mongodb.net` (atlas mode authenticates with the Key Vault
username/password, aks with workload-identity `MONGODB-OIDC`, as
`scripts/seed-workos-org.sh` does from a one-shot pod).

## Prerequisites

Ensure you're connected to the k3d cluster and set the tenant (namespace == database
name) and the MongoDB credentials from the Tilt-created `mongodb-secret`:

```bash
kubectl config use-context k3d-dev
export TENANT=acme
export MONGO_USER=$(kubectl get secret mongodb-secret -n $TENANT -o jsonpath='{.data.username}' | base64 -d)
export MONGO_PASS=$(kubectl get secret mongodb-secret -n $TENANT -o jsonpath='{.data.password}' | base64 -d)
```

`scripts/bootstrap-db.sh` (clear / seed / indexes / verify) and `scripts/create-admin.sh`
script some of the operations below; both still hardcode namespace and database
`devplus`, so edit `NAMESPACE`/`DB_NAME` before using them on a Tilt tenant.

## Accessing MongoDB Shell

### Open an interactive MongoDB shell

```bash
kubectl exec -it -n $TENANT mongodb-0 -- mongosh -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin
```

### Run a one-time MongoDB command

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval "db.adminCommand('ping')"
```

## Database Operations

### List all databases

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval "show dbs"
```

### Switch to the tenant database and list collections

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval "db.getCollectionNames()"
```

### View all modules

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval "db.modules.find().pretty()"
```

### Count modules

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval "db.modules.countDocuments()"
```

### Clear all modules

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval "db.modules.deleteMany({})"
```

### Clear entire database

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval "db.dropDatabase()"
```

## Adding Sample Data

Prefer the UI or `POST /api/v1/modules`, which validates the source and queues the
module for analysis on `module_queue`. A hand-inserted document must use the camelCase
fields of `devplus-shared/models/module.go` (there is no `version` or `provider`
field: the version is the git `ref`, and registry modules carry `registryProvider`).

### Add a single module entry

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval '
db.modules.insertOne({
  sourceType: "git",
  gitUrl: "https://github.com/example/terraform-aws-vpc",
  name: "example-vpc",
  ref: "v1.0.0",
  description: "Example VPC module for AWS",
  parameters: [],
  analysisStatus: "pending",
  createdAt: new Date(),
  updatedAt: new Date()
})
'
```

### Add multiple module entries

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval '
db.modules.insertMany([
  {
    sourceType: "git",
    gitUrl: "https://github.com/example/terraform-aws-vpc",
    name: "example-vpc",
    ref: "v1.0.0",
    description: "VPC module for AWS",
    parameters: [],
    analysisStatus: "pending",
    createdAt: new Date(),
    updatedAt: new Date()
  },
  {
    sourceType: "git",
    gitUrl: "https://github.com/example/terraform-aws-rds",
    name: "example-rds",
    ref: "v2.1.0",
    description: "RDS module for AWS",
    parameters: [],
    analysisStatus: "pending",
    createdAt: new Date(),
    updatedAt: new Date()
  },
  {
    sourceType: "git",
    gitUrl: "https://github.com/example/terraform-gcp-gke",
    name: "example-gke",
    ref: "v1.5.0",
    description: "GKE cluster module for GCP",
    parameters: [],
    analysisStatus: "pending",
    createdAt: new Date(),
    updatedAt: new Date()
  }
])
'
```

## Querying Data

### Find module by name

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval 'db.modules.find({name: "example-vpc"}).pretty()'
```

### Find modules by source type

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval 'db.modules.find({sourceType: "git"}).pretty()'
```

### Find module by ref

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval 'db.modules.find({ref: "v1.0.0"}).pretty()'
```

## Updating Data

### Update a module's description

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval '
db.modules.updateOne(
  {name: "example-vpc"},
  {$set: {description: "Updated VPC module description", updatedAt: new Date()}}
)
'
```

### Update module ref

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval '
db.modules.updateOne(
  {name: "example-vpc"},
  {$set: {ref: "v1.1.0", updatedAt: new Date()}}
)
'
```

## Deleting Data

### Delete a specific module by name

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval 'db.modules.deleteOne({name: "example-vpc"})'
```

### Delete modules by source type

```bash
kubectl exec -n $TENANT mongodb-0 -- mongosh $TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --eval 'db.modules.deleteMany({sourceType: "git"})'
```

## Database Backup and Restore

### Create a backup

```bash
kubectl exec -n $TENANT mongodb-0 -- mongodump --db=$TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin --out=/tmp/backup
kubectl cp $TENANT/mongodb-0:/tmp/backup ./mongodb-backup
```

### Restore from backup

```bash
kubectl cp ./mongodb-backup $TENANT/mongodb-0:/tmp/backup
kubectl exec -n $TENANT mongodb-0 -- mongorestore --db=$TENANT -u "$MONGO_USER" -p "$MONGO_PASS" --authenticationDatabase admin /tmp/backup/$TENANT
```

## Nuclear Option: Reset Everything

### Delete MongoDB persistent storage and restart fresh

```bash
# Delete the PVC (it stays Terminating until the pod releases it)
kubectl delete pvc mongodb-pvc -n $TENANT --wait=false

# Delete the MongoDB pod
kubectl delete pod -n $TENANT -l app=mongodb

# Once the old PVC is gone, recreate it (Tilt's mongodb-pvc resource only creates it when missing)
tilt trigger mongodb-pvc

# Wait for new pod to start
kubectl wait --for=condition=ready pod -l app=mongodb -n $TENANT --timeout=120s
```

Or reset the whole tenant: `tilt down`, `kubectl delete namespace $TENANT`, then `tilt up` again.

## Monitoring

### Check MongoDB pod status

```bash
kubectl get pods -n $TENANT -l app=mongodb
```

### View MongoDB logs

```bash
kubectl logs -n $TENANT -l app=mongodb --tail=50
```

### Follow MongoDB logs in real-time

```bash
kubectl logs -n $TENANT -l app=mongodb --follow
```

### Check MongoDB resource usage

```bash
kubectl top pod -n $TENANT -l app=mongodb
```

## Troubleshooting

### Check PVC status

```bash
kubectl get pvc -n $TENANT
```

### Describe MongoDB pod for events

```bash
kubectl describe pod -n $TENANT -l app=mongodb
```

### Get detailed pod information

```bash
kubectl get pod -n $TENANT -l app=mongodb -o yaml
```

### Check MongoDB connection from backend

```bash
kubectl exec -n $TENANT deployment/backend -- wget -qO- http://mongodb:27017
```
