# UPGRADE FROM OPENEBS 0.8.0 TO 0.8.1

## Overview

This document describes the steps for upgrading OpenEBS from 0.8.0 to 0.8.1 

The upgrade of OpenEBS is a two step process: 
- *Step 1* - Upgrade the OpenEBS Operator 
- *Step 2* - Upgrade the OpenEBS Volumes from previous versions (0.8.0) 

### Terminology
- *OpenEBS Operator : Refers to maya-apiserver & openebs-provisioner along w/ respective services, service a/c, roles, rolebindings*
- *OpenEBS Volume: Storage Engine pods like cStor or Jiva controller(aka target) & replica pods*

## Prerequisites

*All steps described in this document need to be performed on the Kubernetes master or from a machine that has access to Kubernetes master*

### Download the upgrade scripts

The easiest way to get all the upgrade scripts is via git clone.

```
mkdir upgrade-openebs
cd upgrade-openebs
git clone https://github.com/openebs/openebs.git
cd openebs/k8s/upgrade/0.8.0-0.8.1/
```

## Step 1: Upgrade the OpenEBS Operator

### Upgrading OpenEBS Operator CRDs and Deployments

The upgrade steps vary depending on the way OpenEBS was installed, select one of the following:

#### Install/Upgrade using kubectl (using openebs-operator.yaml )

**The sample steps below will work if you have installed openebs without modifying the default values in openebs-operator.yaml. If you have customized it for your cluster, you will have to download the 0.8.1 openebs-operator.yaml and customize it again**

```
# Starting with OpenEBS 0.6, all the components are installed in namespace `openebs`
# as opposed to `default` namespace in earlier releases. 
# If Upgrading from 0.5.x, delete older operator. 
#kubectl delete -f https://raw.githubusercontent.com/openebs/openebs/v0.5/k8s/openebs-operator.yaml
# Wait for objects to be delete, you can check using `kubectl get deploy`

#Upgrade to 0.8.1 OpenEBS Operator
kubectl apply -f https://openebs.github.io/charts/openebs-operator-0.8.1.yaml
```

#### Install/Upgrade using helm chart (using stable/openebs, openebs-charts repo, etc.,) 

**The sample steps below will work if you have installed openebs with default values provided by stable/openebs helm chart.**

- Run `helm repo update` to update local cache with latest package
- Run `helm ls` to get the release name of openebs. 
- Upgrade using `helm upgrade -f https://openebs.github.io/charts/helm-values-0.8.1.yaml <release-name> stable/openebs`

#### Using customized operator YAML or helm chart.
As a first step, you must update your custom helm chart or YAML with 0.8.1 release tags and changes made in the values/templates. 

You can use the following as references to know about the changes in 0.8.1: 
- openebs-charts [PR#2352](https://github.com/openebs/openebs/pull/2352) as reference.

After updating the YAML or helm chart or helm chart values, you can use the above procedures to upgrade the OpenEBS Operator

## Step 2: Upgrade the OpenEBS Pools and Volumes

Even after the OpenEBS Operator has been upgraded to 0.8.1, the cStor Storage Pools and volumes (both jiva and cStor)  will continue to work with older versions. Use the following steps in the same order to upgrade cStor Pools and volumes.

*Note: Upgrade functionality is still under active development. It is highly recommended to schedule a downtime for the application using the OpenEBS PV while performing this upgrade. Also, make sure you have taken a backup of the data before starting the below upgrade procedure.*

Limitations:
- this is a preliminary script only intended for using on volumes where data has been backed-up.
- please have the following link handy in case the volume gets into read-only during upgrade 
  https://docs.openebs.io/docs/next/readonlyvolumes.html
- automatic rollback option is not provided. To rollback, you need to update the controller, exporter and replica pod images to the previous version
- in the process of running the below steps, if you run into issues, you can always reach us on slack


### Upgrade the Jiva based OpenEBS PV 

Extract the PV name using `kubectl get pv`

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                          STORAGECLASS      REASON    AGE
pvc-48fb36a2-947f-11e8-b1f3-42010a800004   5G         RWO            Delete           Bound     percona-test/demo-vol1-claim   openebs-percona             8m
```

```
./jiva_volume_upgrade.sh pvc-48fb36a2-947f-11e8-b1f3-42010a800004
```

### Upgrade cStor Pools

Extract the SPC name using `kubectl get spc`

```
NAME                AGE
cstor-sparse-pool   24m
```

```
./cstor_pool_upgrade.sh cstor-sparse-pool openebs
```
Make sure that this step completes successfully before proceeding to next step.


### Upgrade cStor Volumes

Extract the PV name using `kubectl get pv`

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                  STORAGECLASS           REASON    AGE
pvc-1085415d-f84c-11e8-aadf-42010a8000bb   5G         RWO            Delete           Bound     default/demo-cstor-sparse-vol1-claim   openebs-cstor-sparse             22m
```

```
./cstor_volume_upgrade.sh pvc-1085415d-f84c-11e8-aadf-42010a8000bb openebs
```
