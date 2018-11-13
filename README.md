# Overview

Aerospike is a high performance, flash optimized NoSQL database.

This repo contains the kubectl templates needed to deploy Aerospike EE with Strong Consistency
on GKE. Billing is based on **provisioned** ram/disk. Pricing is available on our 
[Marketplace listing](https://console.cloud.google.com/marketplace/details/aerospike-prod/aerospike-server-enterprise).

The storage engine provided is in-memory with persistence to disk. As such, the capcity requirements
for memory and disk are similar. In this regards, the same value is used to provision both.

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
Subsequent "asinfo" commands occur within the "asadm" prompt from the first line.

1. `kubectl exec ${APP_INSTANCE_NAME}-aerospike-0 -it asadm`

2. `asinfo -v 'roster-set:namespace=${AEROSPIKE_NAMESPACE};nodes=[1,...$AEROSPIKE_NODES]'`  
 eg:  
`asinfo -v 'roster-set:namespace=test;nodes=1,2,3'`

3. `asinfo -v 'recluster:'`


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

Obtain the Custom Application Resource from Google's official marketplace tools repo:
https://github.com/GoogleCloudPlatform/marketplace-k8s-app-tools/tree/master/crd

Apply the CRD:
```
kubectl apply -f app-crd.yaml
```

This command only needs to be ran once per cluster.

### Commands

**Parameters**

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
All `AEROSPIKE_*` parameters except AEROSPIKE\_NODES are optional. Default values are listed above.

All other parameters are required.

**Deploy**

Expand manifest template:
```
cat manifest/* | envsubst > expanded.yaml
```

Run kubectl:
```
kubectl apply -f expanded.yaml --namespace ${NAMESPACE}
```

# Access and Usage

Aerospike is not designed to be publically accessible. Even if you switched the Kubernetes service definition to
load balancer mode, clients will still be unable to connect.

For clients in the same Kubernetes namespace, access Aerospike via its service name: `${APP_INSTANCE_NAME}-aerospike-svc`.


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
kubectl deploy -f expanded-backup.yaml --namespace ${NAMESPACE}
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
kubectl deploy -f expanded-restore.yaml --namespace ${NAMESPACE}
```

After running the above, your cluster that the AEROSPIKE\_SEED\_NODE was part of will contain the data held within your backup volume.

# Scaling

Scaling can be done through the UI or with the following:

```
kubectl scale statefulset ${AEROSPIKE_INSTANCE_NAME}-aerospike --namespace ${NAMESPACE} --replicas [NEW_REPLICA_COUNT]
``` 

Where `[NEW_REPLICA_COUNT]` is the new number of replicas.

Standard pod lifecycle checks will ensure migrations are complete before removing pods. New pods will be added and will be available immediately.

After scaling, you will need to reset your roster:

```
kubectl exec ${APP_INSTANCE_NAME}-aerospike-0 -it asadm

asinfo -v 'roster-set:namespace:${AEROSPIKE_NAMESPACE};nodes=[1,...$AEROSPIKE_NODES]'

asinfo -v 'recluster:'
```

# Upgrades/Updates

As Aerospike is designed as a mission-critical database, upgrades are not applied automatically. This means template updates are not applied immediately,
but rather requires manual pod deletion. Only once the pod has been deleted does the StatefulSet controller redeploy the pod with the new configurations.

It is up to the end-user to ensure that migrations are complete, before continuing to other pods.

Updates and upgrades that do not change cluster size do not need the roster to be reset.

*To update the Aerospike image:*

```
export IMAGE_AEROSPIKE=launcher.gcr.io/aerospike-prod/aerospike-server-enterprise:latest
```

Update the StatefulSet definition with reference to the new image:

```
kubectl patch statefulset ${APP_INSTANCE_NAME}-aerospike \
  --namespace ${NAMESPACE} \
  --type='json' \
  --patch="[{ \
      \"op\": \"replace\", \
      \"path\": \"/spec/template/spec/containers/0/image\", \
      \"value\":\"${IMAGE_AEROSPIKE}\" \
    }]"
```

*To update Aerospike parameters:*

Repeat the same process as creating the application.

```
export AEROSPIKE_NODES=5
export AEROSPIKE_TTL=30
...
cat manifest/aerospike.template.yaml | envsubst > expanded.yaml
kubectl apply -f expanded.yaml --namespace ${NAMESPACE}
```

A reminder, the old pods will still remain in service and the new configuration will be held in reserve and only be applied to new or replacement pods.

# Deletion/Removal

Deletion of the application will remove all pods, service, and jobs. However your persisted data will still exist on the Persistent Volumes created.

Remove the Aerospike application in the UI or with the following:

```
kubectl delete application ${APP_INSTANCE_NAME}
```
To clean up data, you will need to manually remove the persisted volumes. Either do this through the UI or with the following:

```
kubectl delete pvc -l app.kubernetes.io/name=${APP_INSTANCE_NAME} --namespace ${NAMESPACE}
```
