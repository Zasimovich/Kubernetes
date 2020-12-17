
# PV PVC StorageClass

PersistentVolume — непосредственно система хранения, раздел на жёстком диске

PersistentVolumeClaim — запрос от пользователя на использование такого диска
Аналог Kubernetes pod — поды используют ресурсы Worker Node, а PVC — ресурсы PersistentVolume. По аналогии же с подами — поды запрашивают у рабочей ноды ЦПУ и память, а PVC — необходимый им размер диска, и тип доступа — ReadWriteOnce, ReadOnlyMany или ReadWriteMany
'The access modes are:

    ReadWriteOnce -- the volume can be mounted as read-write by a single node
    ReadOnlyMany -- the volume can be mounted read-only by many nodes
    ReadWriteMany -- the volume can be mounted as read-write by many nodes'

```
Reclaim Policy

Current reclaim policies are:

    Retain -- manual reclamation
    Recycle -- basic scrub (rm -rf /thevolume/*)
    Delete -- associated storage asset such as AWS EBS, GCE PD, Azure Disk, or OpenStack Cinder volume is deleted

Currently, only NFS and HostPath support recycling. AWS EBS, GCE PD, Azure Disk, and Cinder volumes support deletion.

```















# POD NODES AFFINITY
[Kubernetes node affinity: Placing pods on specific nodes](https://github.com/Zasimovich/Kubernetes/tree/PV-PVC/StorageClass-PV-PVC)

 - [Node Selector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)

 - [Node Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)
node affinity: Placing pods on specific nodes
 
 - [Pod Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)
 - [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

## Labeling NODES for NODE SELECTOR
- Chose the node/nodes that you wish to run your application on and add a label to it:to it:
```
# kubectl label nodes <node-name> <label-key>=<label-value>
```
- You can choose any key:value pair to mark your nodes. I will use “type:t2medium” as a sample key value pair of my node.

 Now we need to create a pod that gets scheduled to the selected node:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: type
                operator: In
                values:
                - t2medium
      containers:
      - image: nginx
        name: nginx
```
Besides the **requiredDuringSchedulingIgnoredDuringExecution** type of node affinity there exists **preferredDuringSchedulingIgnoredDuringExecution**. The first can be thought of as a “hard” rule, while the second constitutes a “soft” rule that Kubernetes tries to enforce but will not guarantee.

you can indicate that the rule is “soft” rather than a hard requirement, so if the scheduler can’t satisfy it, the pod will still be scheduled.

# Test section for Formatting: URL: https://docs.gitlab.com/ee/user/markdown.html


```diff
- text in red
+ text in green
! text in orange
# text in gray
```


Defferences:
- \# kubectl get pods --all-namespaces -o wide --show-labels

###GIT
Default behavior of “git push” without a branch specified
Command line examples:

To view the current configuration:
\# git config --global push.default

To set a new configuration:
\#git config --global push.default current
```
```