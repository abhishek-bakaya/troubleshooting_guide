
# Manually update MCP (MachineConfigPool)

Node in degraded state because of the use of a deleted machineconfig.

Cluster node is showing `machineconfig.machineconfiguration.openshift.io "rendered-master-xxxx" not found` error message in the node description output.

## Diagnostic Steps

- Check the mcp (MachineConfigPool). The following message shows that the rendered mc (MachineConfig) used is not found.
```
$ oc get mcp
NAME     CONFIG                 UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT
master   rendered-master-xxxx   False      False      True      3              0                   0                     3

$ oc describe mcp master
....
Message:               Node master0.cluster.domain is reporting: "machineconfig.machineconfiguration.openshift.io \"rendered-master-xxxx\" not found"
    Reason:                3 nodes are reporting degraded status on sync
....
```


- Check the logs of the machine-config-operator and machine-config-daemon of the degraded node.
```
$ oc get pods -n openshift-machine-config-operator -o wide
$ oc describe pod machine-config-daemon-tbkwe -n openshift-machine-config-operator
$ oc describe pod machine-config-operator-51b98di73q-bg38i -n openshift-machine-config-operator
```


- Check the node annotations `machineconfiguration.openshift.io/currentConfig` and `machineconfiguration.openshift.io/desiredConfig`. The values should be same and those of the existed machineconfig.
```
$ oc describe node master0.cluster.domain | grep -i config
....
machineconfiguration.openshift.io/currentConfig: rendered-master-xxxx not found
machineconfiguration.openshift.io/desiredConfig: rendered-master-xxxx not found
machineconfiguration.openshift.io/reason: <some reason>
machineconfiguration.openshift.io/state: Degraded
....
```


## Root Cause
The node annotations `machineconfiguration.openshift.io/currentConfig` or `machineconfiguration.openshift.io/desiredConfig` point to a machineConfig that no longer exists, and the `machine-config-operator` can not process that request.


## Resolution
The node annotation `machineconfiguration.openshift.io/desiredConfig` is generated by the `machine-config-controller`, and there is no way to **update** it other than having the controller re-render manually(rendered-config-xxx describes the state of a system). The `machine-config-controller` is not able to understand what the config needs to be since the rendered-config is an aggregate of all machineConfigs assigned to that nodeSelector.

- Assumptions.
```
Working mc: rendered-master-55f7c69f13e362ebcff5f90c7085aa2a
Missing mc: rendered-master-47daa0aaf2aac3477b7dcc774945374a
```


1. Export the existing mc.
```
$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT
master   rendered-master-47daa0aaf2aac3477b7dcc774945374a   False      False      True      3              0                   0                     3

$oc get mc
NAME                                               GENERATEDBYCONTROLLER
....
rendered-master-55f7c69f13e362ebcff5f90c7085aa2a   e54335a3855fc53597e883ba98b548add39a8938
....

$ oc get mc rendered-master-55f7c69f13e362ebcff5f90c7085aa2a -o yaml > mymc.yaml
```


2. Edit mymc.yaml and replace all of the values related to the existing mc with the missing mc values.
```
$ vi mymc.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  annotations:
    machineconfiguration.openshift.io/generated-by-controller-version: e54335a3855fc53597e883ba98b548add39a8938
    machineconfiguration.openshift.io/release-image-version: 4.12.30
  creationTimestamp: "2024-04-01T06:59:36Z"    <----- delete this line
  generation: 1                                <----- delete this line
  name: rendered-master-55f7c69f13e362ebcff5f90c7085aa2a  <----- replace this value with rendered-master-47daa0aaf2aac3477b7dcc774945374a
  ownerReferences:
  - apiVersion: machineconfiguration.openshift.io/v1
    blockOwnerDeletion: true
    controller: true
    kind: MachineConfigPool
    name: master
    uid: 53860789-d7aa-4fbb-a7fc-1ff31b249882  <----- don't delete this
  resourceVersion: "11307"
  uid: f12c80f0-48c8-4003-b25b-3405631dedd4    <----- delete this line
spec:
  baseOSExtensionsContainerImage: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:ebfc5c69cc49f5f1547eecba53ac5bcb9817efc51ecf7e2c608977ea36ff88ae
  config:
....
```


3. Create the missing mc and check if it got created.
```
$ oc create -f mymc.yaml
$ oc get mc | grep rendered-master-47daa0aaf2aac3477b7dcc774945374a
```

If the node is degraded, machine-config-operator won't automatically sync the machineConfig to the degraded node. We have to force it to synchronize.

> ***Note: In case of the degraded mcp of master nodes, do not run following commands in all the master nodes simultaneously. After applying the following process on a master node, wait for it to become `Ready`, then continue on the next master node. In case of degraded mcp of worker nodes, you can run these commands simultaneously on all the worker nodes.***


4. Login to the node, delete the current machineConfig and create a force file.
```
$ oc debug node/master0.cluster.domain
# chroot /host
# rm /etc/machine-config-daemon/currentconfig
# touch /run/machine-config-daemon-force
# exit
# exit
```


5. Edit the node annotations. There are two ways, either automatically by patch command or manually by editing the node.

- By patching.
```
$ oc patch node master0.cluster.domain --type merge --patch "{\"metadata\": {\"annotations\": {\"machineconfiguration.openshift.io/currentConfig\": \"rendered-master-47daa0aaf2aac3477b7dcc774945374a\"}}}"
$ oc patch node master0.cluster.domain --type merge --patch "{\"metadata\": {\"annotations\": {\"machineconfiguration.openshift.io/desiredConfig\": \"rendered-master-47daa0aaf2aac3477b7dcc774945374a\"}}}"
$ oc patch node master0.cluster.domain --type merge --patch '{"metadata": {"annotations": {"machineconfiguration.openshift.io/reason": ""}}}'
$ oc patch node master0.cluster.domain --type merge --patch '{"metadata": {"annotations": {"machineconfiguration.openshift.io/state": "Done"}}}'
```


- Manually replace the value of mc with the newly created one, remove the value of `machineconfiguration.openshift.io/reason` annotation and change the value of `machineconfiguration.openshift.io/state` from `Degraded` to `Done`.
```
$ oc edit node master0.cluster.domain
....
annotations:
....
    machineconfiguration.openshift.io/currentConfig: rendered-master-47daa0aaf2aac3477b7dcc774945374a
    machineconfiguration.openshift.io/desiredConfig: rendered-master-47daa0aaf2aac3477b7dcc774945374a
    machineconfiguration.openshift.io/desiredDrain: uncordon-rendered-master-47daa0aaf2aac3477b7dcc774945374a
    machineconfiguration.openshift.io/lastAppliedDrain: uncordon-rendered-master-47daa0aaf2aac3477b7dcc774945374a
    machineconfiguration.openshift.io/reason: ""
    machineconfiguration.openshift.io/state: Done
```


Check if the mcp is getting updated. If not, then restart the pods of machine-config-operator.
```
$ oc get mcp
$ oc delete pods -n openshift-machine-config-opertor --all
```


A reboot will be triggered and the node will come back to `Ready` state.

