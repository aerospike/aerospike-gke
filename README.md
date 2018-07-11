# Overview

Aerospike is a high performance, flash optimized NoSQL database.

# Installation

## Quick install with Google Cloud Marketplace

Get up and running with a few clicks! Install this Aerospike DB to a
Google Kubernetes Engine cluster using Google Cloud Marketplace. Follow the
on-screen instructions:
*TODO: link to solution details page*

## Command line instructions

Follow these instructions to install Aerospike from the command line.

## Important:

Aerospike is deployed with Strong Consistency enabled. You will need to issue the 
following commands before the cluster is usable.

`kubectl exec ${APP_INSTANCE_NAME}-aerospike-0 -it asadm`

The following commands occur within asadm prompt from above:

`asinfo -v 'roster-set:namespace:${AEROSPIKE_NAMESPACE};nodes=[1,...$AEROSPIKE_NODES]'`  
 eg:  
`asinfo -v 'roster-set:namespace:test;nodes=1,2,3'`

`asinfo -v 'recluster:'`


### Prerequisites

- Setup cluster
- Permissions
- Setup kubectl
- Install Application Resource

*TODO: add details above*

### Commands

Set environment variables (modify if necessary):
```
export APP_INSTANCE_NAME=aerospike-1
export NAMESPACE=default
export IMAGE_AEROSPIKE=launcher.gcr.io/aerospike-prod/aerospike-server:4
export IMAGE_INIT=launcher.gcr.io/aerospike-prod/init:4
export IMAGE_UBBAGENT=launcher.gcr.io/google/ubbagent:4
export AEROSPIKE_NODES=3
```

Expand manifest template:
```
cat manifest/* | envsubst > expanded.yaml
```

Run kubectl:
```
kubectl apply -f expanded.yaml
```

*TODO: fix instructions*

# Backups

Determine the size of your potential backup. Run an estimate by:

`kubectl exec aerospike-1-aerospike-0 asbackup -- --namespace test --estimate`

Where `aerospike-1-aerospike-0` is any deployed pod and `test` is the Aerospike Namespace you've configured.

Based on the above output, provision a volume that's at least 20% larger. Set this value as `BACKUP_SIZE`.
You will need to have a host that's capable of providing that much storage.

Set environment variables (modify if necessary)
```
export AEROSPIKE_SEED_NODE=aerospike-1-aerospike-0
export AEROSPIKE_NAMESPACE=test
export BACKUP_SIZE=4Gi
```

Expand the manifest:
```
envsubst < backup.yaml  > expanded-backup.yaml
```

Run kubectl:
```
kubectl deploy -f expanded-backup.yaml
```

This will result in a persistent volume by the name of "backup-claim". Its contents will be .asb backup files generated by the asbackup utility.


# Restore

Restore assumes you already have a backup volume created from the previous section. If not, simply copy your .asb backup files into a root 
level directory on a volume, and provision said volume as `PersistentVolumeClaim`.

Set environment variables (modify if necessary)
```
export AEROSPIKE_SEED_NODE=aerospike-1-aerospike-0
export AEROSPIKE_NAMESPACE=test
export BACKUP_CLAIM=backup-claim
```

Expand the manifest:
```
envsubst < restore.yaml  > expanded-restore.yaml
```

Run kubectl:
```
kubectl deploy -f expanded-restore.yaml
```

After running the above, your cluster that the AEROSPIKE_SSED_NODE was part of will contain the data held within your backup volume.

# Upgrades/Updates

Aerospike is deployed using a StatefulSet template. As a StatefulSet, the options for a rolling update are either `onDelete` or `rollingUpdate`.
As Aerospike is a database, upgrades are left as `onDelete`, the default `updateStrategy`. This means any template updates are not applied immediately,
but rather requires manual pod deletion. Only once the pod has been deleted does the StatefulSet controller redeploy the pod with the new configurations.

It is up to the end-user to ensure that migrations are complete, before continuing to other pods.
