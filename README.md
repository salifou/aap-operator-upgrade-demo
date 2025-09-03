# AAP OCP Operator Upgrade Demo


Demo of AAP cluster-scoped operator upgrade from version 2.4 to 2.5, leveraging `OperatorGroup.spec.targetNamepaces` to control the upgrade order.

## Links
- [Upgrading AAP Operator from 2.4 to 2.5 on OpenShift](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html-single/installing_on_openshift_container_platform/index#operator-upgrade_licensing-gw)

- [OperatorGroup](https://olm.operatorframework.io/docs/concepts/crds/operatorgroup/)


## High Level Steps

1. Setup:
    - Create demo namespaces (ns1, ns2, ns2)
    - Deploy AAP 2.4 operator in ns1 
    - Deploy AAP 2.4 controllers in ns1 and ns2

2. Backup: Backup AAP controllers

3. Upgrade:
    - Remove all namespaces from the operator group target namespaces => Do not monitor 
    - Upgrade AAP operator from 2.4 to 2.5
    - Create new AAP CR in ns1 and ns2
    - Add ns1 to the operator group target namespaces => UPGRADE
    - Post upgrade tasks

4. Rollback:
    - Remove all namespaces from the operator group target namespaces 
    - Downgrade AAP operator from 2.5 to 2.4
    - Remove AAP CRs, AAP controllers ?
    - Restore AAP controller from backup in step 2.

5. Clean up:
    - Delete all demo namespaces
    - Delete released PV


### 1. Setup

```sh
MANIFEST_DIR=./manifests

# 1. Create demo Namespaces (ns1, ns2 and ns3)
oc apply -f $MANIFEST_DIR/ns1-ns.yml
oc apply -f $MANIFEST_DIR/ns2-ns.yml
oc apply -f $MANIFEST_DIR/ns3-ns.yml

# 2. Deploy AAP 2.4 operator in ns1 namespace watching all namespaces
oc apply -f $MANIFEST_DIR/ns1-og_watch-all.yml
oc apply -f $MANIFEST_DIR/ns1-sub_24.yml

# 3. Approve all install plans in operator namespace if needed
for ip in $(oc get installplan -oname -n ns1); do oc patch $ip --type merge -p '{"spec":{"approved":true}}' -n ns1; done
```

Validate the operator install

```sh
oc -n ns1 get csv

NAME                               DISPLAY                       VERSION              REPLACES                           PHASE
aap-operator.v2.4.0-0.1755833968   Ansible Automation Platform   2.4.0+0.1755833968   aap-operator.v2.4.0-0.1753232791   Succeeded
```

Deploy Automation Controller in ns1 and ns2 namespace.

```sh
# Deploy Automation Controller in ns1 and ns2 namespace.
oc apply -f $MANIFEST_DIR/ns1-ac.yml
oc apply -f $MANIFEST_DIR/ns2-ac.yml
```


### 2. Backup

```sh
# Backup AC deployed in ns1
oc create -f $MANIFEST_DIR/ns1-ac-backup.yml
```


### 3. Upgrade

```sh
#  1. Remove all namespaces from the operator group target namespaces
#    (i.e. point to a non existent ns)
oc apply -f $MANIFEST_DIR/ns1-og_watch-none.yml

# 2. Upgrade AAP operator to 2.5
oc apply -f $MANIFEST_DIR/ns1-sub_25.yml

# 3. Approve all install plans in operator namespace if needed
for ip in $(oc get installplan -oname -n ns1); do oc patch $ip --type merge -p '{"spec":{"approved":true}}' -n ns1; done

# 4. Create AAP CR in ns1 and ns2
oc apply -f $MANIFEST_DIR/ns1-aap.yml
oc apply -f $MANIFEST_DIR/ns2-aap.yml

#  5. Add ns1 to the operator group target namespaces => upgrade
oc apply -f $MANIFEST_DIR/ns1-og_watch-ns1.yml
```

Validation
- AC in ns1 namespace is ugraded
- AC in ns2 namespace is untouched (still 2.4)


### 4. Rollback

TODO

### 5. Clean up

```sh
# 1. Delete namespaces
oc delete -f $MANIFEST_DIR/ns1-ns.yml
oc delete -f $MANIFEST_DIR/ns2-ns.yml
oc delete -f $MANIFEST_DIR/ns3-ns.yml

# 2. Delete released PV
for pv in $(kubectl get pv | grep Released | awk '{print $1}'); do oc delete pv $pv; done
```
