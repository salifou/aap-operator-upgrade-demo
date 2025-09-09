# AAP upgrade Strategies

## Options
1. In Place
1. Side by side same cluster
1. Side by side 2 clusters

## 1. In Place

The cluster scoped operator is upgraded by changing the subscription channel to the desired version (e.g from `Subscription.spec.channel: stable-2.4-cluster-scoped` to `Subscription.spec.channel: stable-2.5-cluster-scoped`)

`OperatorGroup.spec.targetNamespaces` can be used to control the upgrade/roll-out order.

- `OperatorGroup.spec.targetNamespaces)` omitted or empty list => watch all namespaces.
- `OperatorGroup.spec.targetNamespaces: [ns1, ns2]` => monitor ns1 and ns2 namespace only.
- `OperatorGroup.spec.targetNamespaces: [non_existent]` => monitor nothing

We can start with `OperatorGroup.spec.targetNamespaces: [non_existent]` and progressively add namespaces (batch) to be upgraded.

### 1.1 Prerequisites

- A PV for backing up each deployment
- A new DBs for each gateway (new component in 2.5+)

### 1.2 Steps

- Update the OperatorGroup to monitor no namespace (`OperatorGroup.spec.targetNamespaces: [non_existent]`). 
- Backup existing deployments
- Create `AnsibleAutomationPlatform` CR for each deployment
- Upgrade the operator to 2.x by changing `Subscription.spec.channel`
- Approve the install plan if applicable
- Upgrade loop per batch
    - Add batch namespaces to the operator group watch list => upgrade
    - Perform post upgrade tasks
        - users migration
        - validation, etc.

### 1.3 Rollback

Options
1. Using operator provided backup/restore
1. ???

#### 1.3.1 Rollback using operator backup/restore
1. Downgrade AAP operator
1. Restore deployments from backup
1. Delete new `AnsibleAutomationPlatform` CR? ... to be verified

## 2. Side by Side same cluster

The idea is to run both the current and target AAP operator versions in the same cluster.

This can be achieved by:
1. Restricting the existing OperatorGroup target namespaces to namespaces containing AAP components.
1. Replicating the namespaces
1. Deploying a new AAP operator with the target version and targeting replicated namespaces ONLY.

Note: This worked fine in the lab but **NEEDS TO BE REVIEWED AND VALIDATED by the Red Hat OCP SME**.

### 2.1 Prerequisites

- A PV for backing up each existing deployment
- A PV for copying each replica data from backup
    - Can we avoid this ? ... main reasons fow now are:
        - `automationcontrollerrestore.spec.backup_pvc_namespace` is deprecated
        - Restoring from PVC looks at the provided PVC to derive the namespace
- A new DB for each replica
- A new DB for each gateway (new component in 2.5+)
- Enough OCP capacity for running existing and replicated/duplicated deployments at the same time.

### 2.2 Steps

**Preparation:**
- Restrict existing `OperatorGroup` to ONLY watch namespaces containing AAP components.
    => Allow multiple operators with non overlapping target namespaces in same cluster.
- Duplicate existing namespaces
- Deploy a new cluster scoped AAP operator with desired version and OperatorGroup's target namespaces set to duplicated/replicated namespaces.

**Replication Loop for each deployment:**
- Backup existing deployment
- Create a PVC and Copy data from backup
- Restore deployment from backup PVC in replicated namespace

At this point we will have replicated all deployments and can safely upgrade the replicas without impacting customers.

**Upgrade Loop for each replica:**
- See [In-Place upgrade](# 1. In Place)

**Cut-Over:**
- Switch traffic to replicas
    - LB or OCP routes update?
- Decommission old deployments
- Cleanup (unused PVC, DBs, etc.)

### 2.3 Rollback

Nothing beyond cleaning up replicated deployments and resources.
Existing/original deployments are not affected until cut-over.

## 3. Side by Side 2 cluster

The idea is similar to the approach above with key difference being the use of a distinct OCP cluster for replication.

### 3.1 Prerequisites

- A separate cluster with enough resource for the replica.
- A separate DB for each replica
- A DB for each gateway component

### 3.2 Prerequisites

TODO

### 3.3 Rollback

Nothing beyond cleaning up replicated deployments and resources. 
Existing/original deployments are not affected until cut-over.

