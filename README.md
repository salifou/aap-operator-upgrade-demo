# Demo

AAP cluster scoped operator upgrade minimizing downtime.

This is achieved by:
- Having 2 operator instances and watching non overlapping namespace sets (set 1 & 2).
- Replicate existing deployments in the second namespace set.
- Upgrade the replicas.
- Cut-over and remove orginal deployments if successful.


## Initial State

- 2 Namespaces: ns1, ns2 (first namespace set)
- 1 Cluster scoped AAP operator 2.4 installed in ns1
- `AutomationController` deployed in ns1 and ns2


## Steps

1. Configure the operator to watch the first namespace set **ONLY**
2. Create a second namespace set: rs1, rs2
3. Deploy a second 2.4 operator in rs1 watching second namespace set **ONLY**
4. Backup automation controller in ns1
5. Restore automation controller in rs1
6. Upgrade automation controller in rs1 to **2.5**
7. Cut-over & remove original deployments


### 1. Configure the operator to watch the first namespace set

```sh
oc apply -f manifests/op1-og-set1.yml
```

### 2. Create a second namespace set: rs1, rs2

```sh
oc apply -f manifests/ns-set2.yml
```

### 3. Deploy a second 2,4 operator in rs1 watching second namespace set

```sh
### a. Install AAP 2.4 operator
oc apply -f manifests/op2-og-set2.yml
oc apply -f manifests/op2-sub-24.yml

### b. Approve install plan
for ip in $(oc get ip -oname -n rs1); do oc patch $ip --type merge -p '{"spec":{"approved":true}}' -n rs1; done

### c. Wait for install to complete
CSV=$(oc -n rs1 get csv -oname)
oc -n rs1 wait $CSV --for jsonpath='{.status.phase}'=Succeeded --timeout 5m

### d. Verify the install
oc -n rs1 get csv

NAME                               DISPLAY                       VERSION              REPLACES                           PHASE
aap-operator.v2.4.0-0.1755833968   Ansible Automation Platform   2.4.0+0.1755833968   aap-operator.v2.4.0-0.1753232791   Succeeded
```

### 4. Backup automation controller in ns1

```sh
### a. backup AC in ns1
oc apply -f manifests/ns1-ac-backup.yml

### b. Wait for the backup to complete
oc -n ns1 wait automationcontrollerbackup/dev-ac-backup-1 --for condition=Successful --timeout 5m
```

### 5. Restore automation controller in rs1

```sh
### a. Copy backup data from ns1
oc apply -f manifests/data-copy/ns1-data-copy-pod.yml
oc -n ns1 rsync data-copy:/data/ manifests/data-copy/data/

### b. Create a PVC in rs1 and copy backup data
oc apply -f manifests/data-copy/rs1-data-copy-pod.yml
oc -n rs1 rsync manifests/data-copy/data/ data-copy:/data/

### c. Restore backp from PVC in rs1
oc apply -f manifests/rs1-ac-restore.yml
oc -n rs1 wait automationcontrollerrestore/dev-ac-restore-1 --for condition=Successful --timeout 20m

### d. Wait for AC deployment to complete
oc -n rs1 wait automationcontroller/dev --for condition=Successful --timeout 10m
```

**Notes**
- May need to disable Scheduled Job to avoid duplicate

### 6. Upgrade automation controller in rs1 to 2.5

See: [AAP Upgrade on OpenShift](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html-single/installing_on_openshift_container_platform/index#operator-upgrade_licensing-gw)

```sh
### a. Create AAP CRD
oc apply -f manifests/rs1-aap.yml

### b. Upgrade the operator channel to 2.5 => UPGRADE
oc apply -f manifests/op2-sub-25.yml

### c. Approve install plan if applicable
for ip in $(oc get ip -oname -n rs1); do oc patch $ip --type merge -p '{"spec":{"approved":true}}' -n rs1; done

### d. Wait for upgrade to finish
oc -n rs1 wait ansibleautomationplatform/myaap --for condition=Successful --timeout 20m

### e. Perform post upgrade tasks
#      - user migration, validation, etc.
```

### 7. Cut-over & remove original deployments

- Switch traffic to replicas
    Update OCP routes or Reverse Proxy?
- Decommission old deployments
- Cleanup (unused PVC, DBs, etc.)


## Links

- [Upgrading AAP Operator from 2.4 to 2.5 on OpenShift](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html-single/installing_on_openshift_container_platform/index#operator-upgrade_licensing-gw)

- [OperatorGroup](https://olm.operatorframework.io/docs/concepts/crds/operatorgroup/)