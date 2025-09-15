# AAP upgrade Strategies

## Options
1. In Place
1. Side by side Two clusters
1. Side by side One cluster

### 1. In Place

This is the standard approach where the channel is updated to the desired version (e.g. from `stable-2.4-cluster-scoped` to `stable-2.5-cluster-scoped`). This will trigger the upgrade of components managed by the operator (AC, HUB, EDA, etc.). See: [Upgrading AAP operator on OpenShift](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html-single/installing_on_openshift_container_platform/index#operator-upgrade_licensing-gw)

Downtime is inevitable and no control of namespaces upgrade order by default.

`OperatorGroup.spec.targetNamespaces` can be used to control namespace upgrade order by starting with no namespace and progressively namespaces to be upgraded.


### 2. Side by Side Two clusters

We start with 2 OpenShift clusters, AAP components deployed in the first cluster.

1. Replicate AAP deployments in the second cluster
1. Upgrade replicas
1. Cut-over and decommission original deployments successful.

This approach minimizes downtime by keeping original deployments untouched and running until cut-over.

### 3. Side by Side One cluster

We start with a single OCP cluster, one AAP operator instance and AAP deployments in multiple namespaces (set 1)

1. Configure the operator to monitor the first namespace set ONLY.
1. Create a second namespace set
1. Deploy a second operator instance monitoring the second namespace set
1. Replicate the AAP deployments in the second namespace set
1. Upgrade the replicas (Standard approach)
1. Cut-over and decommission original deployments 

`OperatorGroup.spec.targetNamespaces` is used to configure the list of namespace monitored by the opertor.


## Prerequisites

- AAP operator 2.4 and 2.x available in the catalog (available for install)
- Persistent Storage
	- 1 PV for each AAP deployment backup
	- 1 PV for each Replica (1)
- Database
	- 1 DB for each new gateway component
	- 1 DB for each replica (1)
- Enough capacity for running original & replicas (1)

(1) Side-by-side only

## Prep tasks

- Prerequisites
- Review AAP Backup/Restore
- Step by step upgrade process
- Test and validation
- Team dependencies & responsibilities
- Upgrade automation/scripts
  - Step/Script to disable/enable scheduled jobs (1)
  - etc.
- Customization (nginx, redis, etc.) & Potential Impact
- Customers
  - REST endpoints and payload
  - Changes needed
  - Communication
- Sizing of new components

(1) Side-by-side only

## Rollback

### In Place
- [operator backup/restore]: Downgrade operator and restore from backup.
- [DB snapshot]: ?

## Side by Side
Nothing beyond cleaning up replicated deployment and resources.
