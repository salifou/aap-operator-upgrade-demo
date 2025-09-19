# AAP upgrade Strategies

## Options
1. In Place
1. Side by side Two clusters
1. Side by side One cluster

### 1. In Place

This is the standard approach where the channel is updated to the desired version (e.g. from `stable-2.4-cluster-scoped` to `stable-2.5-cluster-scoped`). This will trigger the upgrade of components managed by the operator (AC, HUB, EDA, etc.). See: [Upgrading AAP operator on OpenShift](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html-single/installing_on_openshift_container_platform/index#operator-upgrade_licensing-gw)

Downtime is inevitable (upgrade & rollback) and no control of namespaces upgrade order by default.

`OperatorGroup.spec.targetNamespaces` can be used to control namespace upgrade order.
This is achieved by starting with an empty list (no namespace) and progressively adding namespaces to be upgraded to the list.


### 2. Side by Side Two clusters

We start with 2 OpenShift clusters, a single cluster scoped AAP operator in each OCP cluster, multiple AAP deployments/namespaces in the first cluster.

1. Replicate namespaces and AAP deployments in the second cluster
1. Upgrade replicas
1. Cut-over and decommission original deployments if successful.

This approach minimizes downtime by keeping original deployments untouched and running until cut-over.

### 3. Side by Side One cluster

We start with a single OCP cluster, one cluster scoped AAP operator, multiple AAP deployments/namespaces (set 1)

1. Configure the operator to monitor the first namespace set **ONLY**.
1. Create a second namespace set
1. Deploy a second operator instance monitoring the second namespace set **ONLY**
1. Replicate the AAP deployments in the second namespace set
1. Upgrade the replicas (Standard approach)
1. Cut-over and decommission original deployments 

`OperatorGroup.spec.targetNamespaces` is used to configure the list of namespace monitored by the AAP operator.

