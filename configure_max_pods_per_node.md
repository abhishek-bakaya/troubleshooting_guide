# Manage maximum number of pods per node

In OpenShift Container Platform, you can configure the number of pods that can run on a node based on the number of processor cores on the node, a hard limit or both. Two parameters control the maximum number of pods that can be scheduled to a node: `podsPerCore` and `maxPods`.  If you use both options, the lower of the two limits the number of pods on a node.


- Obtain the label associated with the MachineConfigPool.
```
$ oc edit mcp master
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  creationTimestamp: "2024-02-26T10:53:26Z"
  generation: 5
  labels:
    pools.operator.machineconfiguration.openshift.io/master: ""
  name: master
....
```


- Create a custom resource (CR) for your configuration change.
```
$ vi pods-limit-config.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: set-max-pods-limit
spec:
  machineConfigPoolSelector:
    matchLabels:
      pools.operator.machineconfiguration.openshift.io/master: ""
  kubeletConfig:
    podsPerCore: 10
    maxPods: 320
```


- Create the CR.
`$ oc create -f pods-limit-config.yaml`


- Check if the changes are taking place.
```
$ oc get mcp

`UPDATING` column should be reporting `TRUE`
```

A reboot will get triggered.
