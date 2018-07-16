# Overview

Aerospike is a high performance, flash optimized NoSQL database.

This repo contains the kubectl templates needed to deploy Aerospike EE with Strong Consistency
on GKE. Billing is based on **provisioned** ram/disk. Pricing is available on our 
[Marlekplace listing](https://console.cloud.google.com/marketplace/details/aerospike-prod/aerospike-server-enterprise).

The storage engine provided is in-memory with persistence to disk. As such, the capcity requirements
for memory and disk are similar. In this regards, the same value is used to provision both. See 
`schema.yaml` for the list of parameters.

# Installation

## Quick install with Google Cloud Marketplace

Get up and running with a few clicks! Install this Aerospike DB to a
Google Kubernetes Engine cluster using Google Cloud Marketplace. Follow the
on-screen instructions:

https://console.cloud.google.com/launcher/kubernetes/config/aerospike-prod/aerospike-server-enterprise

## Command line instructions

Follow these instructions to install Aerospike from the command line.

## Important:

**Cluster Roster**

Aerospike is deployed with Strong Consistency enabled. You will need to issue the 
following commands to set the roster before the cluster is usable.
Subsequent "asinfo" commands occur within asadm prompt from the first line.

1. `kubectl exec ${APP_INSTANCE_NAME}-aerospike-0 -it asadm`

2. `asinfo -v 'roster-set:namespace:${AEROSPIKE_NAMESPACE};nodes=[1,...$AEROSPIKE_NODES]'`  
 eg:  
`asinfo -v 'roster-set:namespace:test;nodes=1,2,3'`

3. `asinfo -v 'recluster:'`

**Authentication**

Aerospike is deployed with default credentials of `admin/admin'. Aerospike highly recommends you change 
the password before entering production by using the following command:  
`kubectl exec ${APP_INSTANCE_NAME} aql -- -c "set password ${NEWPASSWORD} for admin"`



### Prerequisites

**Setup cluster**

You should already have a kubernetes cluster provisioned. If you do not have a cluster, please follow 
[these instructions](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster) on setting up a GKE cluster. 

Alternatively, you can use the following CLI commands.

Log in as yourself by running:

```
gcloud auth login
```

Provision a GKE cluster and configure kubectl to connect to it:
```
CLUSTER=cluster-1
ZONE=us-west1-a

# Create the cluster.
gcloud beta container clusters create "$CLUSTER" \
    --zone "$ZONE" \
    --machine-type "n1-standard-1" \
    --num-nodes "3"

# Configure kubectl authorization.
gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"

# Bootstrap RBAC cluster-admin for your user.
# More info: https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin --user $(gcloud config get-value account)

# (Optional) Start up kubectl proxy.
kubectl proxy
```
**License Secret**

You must obtain a license secret from GCP Marketplace to launch this application.
You can obtain the license from the listing page: https://console.cloud.google.com/marketplace/details/aerospike-prod/aerospike-server-enterprise.

[TODO](#HowDoUsersGetALicense)

The license secret is a Kubernetes Secret. Keep the name of this secret handy for the following section.

**Install Application Resource**

*TODO: add details above*

### Commands


Set environment variables (modify if necessary):
```
export APP_INSTANCE_NAME=aerospike-1
export NAMESPACE=default
export REPORTING_SECRET=aerospike-1-reporting-secret
export IMAGE_AEROSPIKE=launcher.gcr.io/aerospike-prod/aerospike-server:4
export IMAGE_INIT=launcher.gcr.io/aerospike-prod/init:4
export IMAGE_UBBAGENT=launcher.gcr.io/google/ubbagent:4
export AEROSPIKE_NODES=3
export AEROSPIKE_NAMESPACE=test
export AEROSPIKE_REPL=2
export AEROSPIKE_MEM=1
export AEROSPIKE_TTL=0
```
All `AEROSPIKE_*` parameters except AEROSPIKE_NODES are optional. Default values can be found at:

https://github.com/aerospike/aerospike-server.docker/blob/master/README.md#custom-configuration

All other parameters are required.

Expand manifest template:
```
cat manifest/* | envsubst > expanded.yaml
```

Run kubectl:
```
kubectl apply -f expanded.yaml
```


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

After running the above, your cluster that the AEROSPIKE_SEED_NODE was part of will contain the data held within your backup volume.

# Scaling

Scaling can be done through the UI or with the following:

```
kubectl scale statefulset ${AEROSPIKE_INSTANCE_NAME}-aerospike --replicas ${NEW_REPLICA_COUNT}
```

Standard pod lifecycle checks will ensure migrations are complete before removing pods. New pods will be added and will be available immediately.

After scaling, you will need to reset your roster:

```
kubectl exec ${APP_INSTANCE_NAME}-aerospike-0 -it asadm

asinfo -v 'roster-set:namespace:${AEROSPIKE_NAMESPACE};nodes=[1,...$AEROSPIKE_NODES]'

asinfo -v 'recluster:'
```

# Upgrades/Updates

Aerospike is deployed using a StatefulSet template. As a StatefulSet, the options for a rolling update are either `onDelete` or `rollingUpdate`.
As Aerospike is a database, upgrades are left as `onDelete`, the default `updateStrategy`. This means any template updates are not applied immediately,
but rather requires manual pod deletion. Only once the pod has been deleted does the StatefulSet controller redeploy the pod with the new configurations.

It is up to the end-user to ensure that migrations are complete, before continuing to other pods.

Updates and upgrades that do not change cluster size do not need the roster to be reset.

# Deletion/Removal

Deletion of the application will remove all pods, service, and jobs. However your persisted data will still exist on the Persistent Volumes created.

Remove the Aerospike application in the UI or with the following:

```
kubectl delete application ${APP_INSTANCE_NAME}
```
To clean up data, you will need to manually remove the persisted volumes. Either do this through the UI or with the following:

```
kubectl delete pvc -l app.kubernetes.io/name=${APP_INSTANCE_NAME}
```
