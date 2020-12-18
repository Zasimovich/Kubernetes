
*Для кластера в Амазоне достаточно создать StorageClass и PVC который к нему обращается, чтобы заказать для пода файловое хранилище.  
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
PersistentVolume могут быть созданы двумя способами — динамически (рекомендуемый способ), и статически.

При статичном методе сначала создаётся набор дисков, например EBS, которые затем используются кластером для PersistentVolumeClaim.

В случе, если для PersistentVolumeClaim не удалось найди подходящий PV — кластер может создать отдельный PV конкретно для этого PVC — это и будет динамическое создание PVC.

При этом в PVC должен быть задан Storage Class, и такой класс должен поддерживаться кластером.


## Типы дисков

Для понимания роли PersistentVolume — рассмотрим доступные системы хранения:

    Node-local хранение (emptyDir и hostPath) - Хранят данные на диске Ноды на котором запущен под. Даже если под перезапустится, данные с ноды не удалятся. Удалятся они только в случае удаления пода в ручную.
    Cloud volumes (например, awsElasticBlockStore, gcePersistentDisk и azureDiskVolume)
    File-sharing volumes, такие как Network File System
    Distributed-file systems (например, CephFS, RBD и GlusterFS)
    специальные типы разделов, такие как PersistentVolumeClaim, secret, ConfigMap и gitRepo

# Создание AWS EBS и привязываение его в ручную к PV
\# aws ec2 --profile arseniy --region us-east-2 create-volume --availability-zone us-east-2a --size 50
Мы получаем vol-ID типа - "VolumeId": "vol-0928650905a2491e2"
И далее создаем PV

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-static
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  awsElasticBlockStore:
    fsType: ext4
    volumeID: vol-0928650905a2491e2
```
- тут ReadWriteOnce — раздел может быть смонтирован только к одной рабочей ноде с правами чтения/записи

## StorageClass
Параметр storageClassName определяет тип хранилища.
И для PVC, и для PV должен быть задан один и тот же класс, иначе PVC не подключит PV, и STATUS такого PVC будет Pending.
Если для PVC не задан StorageClass — будет использован дефолтный:

## Создание PersistentVolumeClaim
Добавляем PersistentVolumeClaim, который будет использовать этот PV:

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-static
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeName: pv-static
  ```

  Модем создать PVC который вместо указания и привязки к PV, будет использовать StorageClass

  ```
  ---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fileshare
  labels:
    app: fileshare
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: defaul
  ```

  ## Dynamic PersistentVolume provisioning

Динамическое создание PersistentVolume аналогично статическому с той разницей, что мы не создаём EBS, и не создаём отдельного ресурса PersistentVolume — вместо этого мы опишем PersistentVolumeClaim, который самостоятельно создаст EBS, и смонтирует его к EC2 WorkerNode


Команды дебага:
\# kubectl describe pvc pvc-dynamic
\# kubectl describe pv pvc-6d024b40-a239-4c35-8694-f060bd117053
\# aws ec2 --profile arseniy --region us-east-2 describe-volumes --volume-ids vol-040a5e004876f1a40 --output json
Определяем, в какой AvailabilityZone расположен наш EBS:
\# aws ec2 --profile arseniy --region us-east-2 describe-volumes --volume-ids vol-0928650905a2491e2 --query '[Volumes[*].AvailabilityZone]'  --output text
us-east-2a
us-east-2a — окей, значит, нам надо и под создавать в той же AvailabilityZone.

Пишем манифест для пода:
```
apiVersion: v1
kind: Pod
metadata:
  name: pv-static-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: failure-domain.beta.kubernetes.io/zone
            operator: In
            values:
            - us-east-2a
  volumes:
    - name: pv-static-storage
      persistentVolumeClaim:
        claimName: pvc-static
  containers:
    - name: pv-static-container
      image: nginx
      ports:
        - containerPort: 80
          name: "nginx"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-static-storage
```
С помощью nodeAffinity  мы привязываем под к той ноде, которая находится в соответствующей AvailabilityZone в которой находится и нах PVC, чтобы он мог подключится к нашей ноде(поду) 

В отличии от Dynamic PVC — тут через nodeAffinity мы явно задаём поиск ноды для этого пода в зоне us-east-2a.

## Удаление PersistentVolume и PersistentVolumeClaim

Когда пользователь удаляет PVC, который используется подом, этот PVC удаляется не сразу — удаление откладывается, пока не будет остановлен использующий его под.

Аналогично, при удалении PV, к которому есть binding от какого-либо PVC, этот PV тоже удаляется не сразу, до тех пор, пока существует связь между PVC и PV.

## Reclaiming
Когда пользователь заканчивает работу с PersistentVolume, он может удалить его объект из кластера, что бы освободить ресурс AWS EBS (reclaim).

Reclaim policy для PersistentVolume указывает кластеру, что делать с овободившимся диском, и может иметь значение Retained, Recycled или Deleted.
Retain

Retain политика позволяет выполнять ручную очистку диска.

После удаления соответвующего PersistentVolumeClaim, PersistentVolume остаётся, и отмечается как «released«, однако становится недоступен для новых PersistentVolumeClaim, т.к. содержит данные предыдущего PersistentVolumeClaim.

Для того, что бы использовать такой ресурс повторно — можно либо удалить объект PersistentVolume, при этом AWS EBS останется доступен, био вручную удалить данные с диска.
Delete

При Delete — удаление PVC приводит к удалению и соответствующего устройсва, такого как AWS EBS, GCE PD или Azure Disk.

Разделы, созданные автоматически наследуют политику из StorageClass, которая по умолчанию задана в Delete.
Recycle

Устарела. Выполняет удаление с раздела обычным rm -rf.




Проверим в самом поде:
\# kk exec -ti pv-dynamic-pod bash
```
root@pv-dynamic-pod:/# lsblk
```
nvme1n1 — наш диск.

## Как пропатчить PVчтобы изменить RECLAIM POLICY

Смотрим 
\# kubectl get pv
или
\# kubectl get pv pv-static -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
Патчим:
\# kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
или на Retain
\# kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'





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
  You can indicate that the rule is “soft” rather than a hard requirement, so if the scheduler can’t satisfy it, the pod will still be scheduled.





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