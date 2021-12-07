# Autoscaling 

Horizontal Pod Autoscaling (HPA) 可以根据 CPU 使用率或应用自定义 metrics 自动扩展 Pod 数量（支持 replication controller、deployment 和 replica set ）。

* 控制管理器每隔 30s（可以通过 --horizontal-pod-autoscaler-sync-period 修改）查询 metrics 的资源使用情况
* 支持三种 metrics 类型
  * 预定义 metrics（比如 Pod 的 CPU）以利用率的方式计算
  * 自定义的 Pod metrics，以原始值（raw value）的方式计算
  * 自定义的 object metrics
* 支持两种 metrics 查询方式：Heapster 和自定义的 REST API
* 支持多 metrics


# 容器类


# 存储类


# 服务类


## Service 

定义了一个名为 nginx 的服务，将服务的 80 端口转发到 default namespace 中带有标签 run=nginx 的 Pod 的 80 端口

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: ClusterIP
```

自定义 endpoint，即创建同名的 service 和 endpoint，在 endpoint 中设置外部服务的 IP 和端口

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
kind: Endpoints
apiVersion: v1
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 1.2.3.4
    ports:
      - port: 9376
```

通过 DNS 转发，在 service 定义中指定 externalName。此时 DNS 服务会给 <service-name>.<namespace>.svc.cluster.local 创建一个 CNAME 记录，其值为 my.database.example.com。并且，该服务不会自动分配 Cluster IP，需要通过 service 的 DNS 来访问。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: default
spec:
  type: ExternalName
  externalName: my.database.example.com
```

不需要 Cluster IP 的服务，通过 <service-name>.<namespace>.svc.cluster.local 访问服务，DNS请求的结果实际为Pod IP地址。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  clusterIP: None
  ports:
  - name: tcp-80-80-3b6tl
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  replicas: 2
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: Always
        name: nginx
        resources:
          limits:
            memory: 128Mi
          requests:
            cpu: 200m
            memory: 128Mi
      dnsPolicy: ClusterFirst
      restartPolicy: Always
```



## Ingress

单服务的 Ingress 即该 Ingress 仅指定一个没有任何规则的后端服务。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
```

多服务的 Ingress

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```

虚拟主机 Ingress 即根据名字的不同转发到不同的后端服务上，而他们共用同一个的 IP 地址

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: s1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: s2
          servicePort: 80
```

TLS Ingress 通过 Secret 获取 TLS 私钥和证书 (名为 tls.crt 和 tls.key)，来执行 TLS 终止。

```yaml
apiVersion: v1
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
kind: Secret
metadata:
  name: testsecret
  namespace: default
type: Opaque
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: no-rules-map
spec:
  tls:
    - secretName: testsecret
  backend:
    serviceName: s1
    servicePort: 80
```



# 配置类


## ConfigMap

在执行应用程式或是生产环境等等, 会有许多的情况需要做变更, 而我们不希望因应每一种需求就要准备一个镜像档, 这时就可以透过 ConfigMap 来帮我们做一个配置档或是命令参数的映射, 更加弹性化使用我们的服务或是应用程式。

ConfigMap 用于保存配置数据的键值对，可以用来保存单个属性，也可以用来保存配置文件。ConfigMap 跟 secret 很类似，但它可以更方便地处理不包含敏感信息的字符串。


# CronJob

CronJob 即定时任务，就类似于 Linux 系统的 crontab，在指定的时间周期运行指定的任务。


# CustomResourceDefinition

CustomResourceDefinition（CRD）是 v1.7 新增的无需改变代码就可以扩展 Kubernetes API 的机制，用来管理自定义对象。它实际上是 ThirdPartyResources（TPR）的升级版本，而 TPR 已经在 v1.8 中弃用。


# DaemonSet

DaemonSet 保证在每个 Node 上都运行一个容器副本，常用来部署一些集群的日志、监控或者其他系统管理应用。典型的应用包括：

* 日志收集，比如 fluentd，logstash 等
* 系统监控，比如 Prometheus Node Exporter，collectd，New Relic agent，Ganglia gmond 等
* 系统程序，比如 kube-proxy, kube-dns, glusterd, ceph 等


# Deployment

Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义 (declarative) 方法，用来替代以前的 ReplicationController 来方便的管理应用。

```yaml

```
## 


# Job

Job 负责批量处理短暂的一次性任务 (short lived one-off tasks)，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。

Kubernetes 支持以下几种 Job：

* 非并行 Job：通常创建一个 Pod 直至其成功结束
* 固定结束次数的 Job：设置 .spec.completions，创建多个 Pod，直到 .spec.completions 个 Pod 成功结束
* 带有工作队列的并行 Job：设置 .spec.Parallelism 但不设置 .spec.completions，当所有 Pod 结束并且至少一个成功时，Job 就认为是成功


## Job Controller

Job Controller 负责根据 Job Spec 创建 Pod，并持续监控 Pod 的状态，直至其成功结束。如果失败，则根据 restartPolicy（只支持 OnFailure 和 Never，不支持 Always）决定是否创建新的 Pod 再次重试任务。


# 本地数据卷

本地数据卷（Local Volume）代表一个本地存储设备，比如磁盘、分区或者目录等。主要的应用场景包括分布式存储和数据库等需要高性能和高可靠性的环境里。本地数据卷同时支持块设备和文件系统，通过 spec.local.path 指定；但对于文件系统来说，kubernetes 并不会限制该目录可以使用的存储空间大小。

本地数据卷只能以静态创建的 PV 使用。相对于 HostPath，本地数据卷可以直接以持久化的方式使用（它总是通过 NodeAffinity 调度在某个指定的节点上）。


# Namespace

Namespace 是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。常见的 pod, service, replication controller 和 deployment 等都是属于某一个 namespace 的（默认是 default），而 node, persistent volume，namespace 等资源则不属于任何 namespace。

Namespace 常用来隔离不同的用户，比如 Kubernetes 自带的服务一般运行在 kube-system namespace 中。


# Network Policy

随着微服务的流行，越来越多的云服务平台需要大量模块之间的网络调用。Kubernetes 在 1.3 引入了 Network Policy，Network Policy 提供了基于策略的网络控制，用于隔离应用并减少攻击面。它使用标签选择器模拟传统的分段网络，并通过策略控制它们之间的流量以及来自外部的流量。

在使用 Network Policy 时，需要注意

* v1.6 以及以前的版本需要在 kube-apiserver 中开启 extensions/v1beta1/networkpolicies
* v1.7 版本 Network Policy 已经 GA，API 版本为 networking.k8s.io/v1
* v1.8 版本新增 Egress 和 IPBlock 的支持
* 网络插件要支持 Network Policy，如 Calico、Romana、Weave Net 和 trireme 等


## Namespace 隔离

默认情况下，所有 Pod 之间是全通的。每个 Namespace 可以配置独立的网络策略，来隔离 Pod 之间的流量。

v1.7 + 版本通过创建匹配所有 Pod 的 Network Policy 来作为默认的网络策略，比如默认拒绝所有 Pod 之间 Ingress 通信。

默认拒绝所有 Pod 之间 Egress 通信的策略为

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

默认拒绝所有 Pod 之间 Ingress 和 Egress 通信的策略为

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

默认允许所有 Pod 之间 Ingress 通信的策略为

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  ingress:
  - {}
```

默认允许所有 Pod 之间 Egress 通信的策略为

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  egress:
  - {}
```

而 v1.6 版本则通过 Annotation 来隔离 namespace 的所有 Pod 之间的流量，包括从外部到该 namespace 中所有 Pod 的流量以及 namespace 内部 Pod 相互之间的流量


## Pod 隔离

通过使用标签选择器（包括 namespaceSelector 和 podSelector）来控制 Pod 之间的流量。


比如下面的 Network Policy：

* 允许 default namespace 中带有 role=frontend 标签的 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口
* 允许带有 project=myprojects 标签的 namespace 中所有 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口

```yaml
# v1.6 以及更老的版本应该使用 extensions/v1beta1
# apiVersion: extensions/v1beta1
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: tcp
      port: 6379
```

隔离 default namespace 中带有 role=db 标签的 Pod：

* 允许 default namespace 中带有 role=frontend 标签的 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口
* 允许带有 project=myprojects 标签的 namespace 中所有 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口
* 允许 default namespace 中带有 role=db 标签的 Pod 访问 10.0.0.0/24 网段的 TCP 5987 端口

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```


# Node

Node 是 Pod 真正运行的主机，可以是物理机，也可以是虚拟机。为了管理 Pod，每个 Node 节点上至少要运行 container runtime（比如 docker 或者 rkt）、kubelet 和 kube-proxy 服务。

![node](images/node.png "node")


## Node 管理

不像其他的资源（如 Pod 和 Namespace），Node 本质上不是 Kubernetes 来创建的，Kubernetes 只是管理 Node 上的资源。虽然可以通过 Manifest 创建一个 Node 对象（如下 yaml 所示），但 Kubernetes 也只是去检查是否真的是有这么一个 Node，如果检查失败，也不会往上调度 Pod。

```yaml
kind: Node
apiVersion: v1
metadata:
  name: 10-240-79-157
  labels:
    name: my-first-k8s-node
```

这个检查是由 Node Controller 来完成的。Node Controller 负责：

* 维护 Node 状态
* 与 Cloud Provider 同步 Node
* 给 Node 分配容器 CIDR
* 删除带有 NoExecute taint 的 Node 上的 Pods

默认情况下，kubelet 在启动时会向 master 注册自己，并创建 Node 资源。


## Node 的状态

每个 Node 都包括以下状态信息：

* 地址：包括 hostname、外网 IP 和内网 IP
* 条件（Condition）：包括 OutOfDisk、Ready、MemoryPressure 和 DiskPressure
* 容量（Capacity）：Node 上的可用资源，包括 CPU、内存和 Pod 总数
* 基本信息（Info）：包括内核版本、容器引擎版本、OS 类型等


## Taints 和 tolerations

Taints 和 tolerations 用于保证 Pod 不被调度到不合适的 Node 上，Taint 应用于 Node 上，而 toleration 则应用于 Pod 上（Toleration 是可选的）。

比如，可以使用 taint 命令给 node1 添加 taints：

```bash
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value2:NoExecute
```


## Node 维护模式

标志 Node 不可调度但不影响其上正在运行的 Pod，这种维护 Node 时是非常有用的

```bash
kubectl cordon $NODENAME
```


# Persistent Volume

PersistentVolume (PV) 和 PersistentVolumeClaim (PVC) 提供了方便的持久化卷：PV 提供网络存储资源，而 PVC 请求存储资源。这样，设置持久化的工作流包括配置底层文件系统或者云数据卷、创建持久性数据卷、最后创建 PVC 来将 Pod 跟数据卷关联起来。PV 和 PVC 可以将 pod 和数据卷解耦，pod 不需要知道确切的文件系统或者支持它的持久化引擎。


## Volume 生命周期

Volume 的生命周期包括 5 个阶段

* Provisioning，即 PV 的创建，可以直接创建 PV（静态方式），也可以使用 StorageClass 动态创建
* Binding，将 PV 分配给 PVC
* Using，Pod 通过 PVC 使用该 Volume，并可以通过准入控制 StorageObjectInUseProtection（1.9 及以前版本为 PVCProtection）阻止删除正在使用的 PVC
* Releasing，Pod 释放 Volume 并删除 PVC
* Reclaiming，回收 PV，可以保留 PV 以便下次使用，也可以直接从云存储中删除
* Deleting，删除 PV 并从云存储中删除后段存储

根据这 5 个阶段，Volume 的状态有以下 4 种

* Available：可用
* Bound：已经分配给 PVC
* Released：PVC 解绑但还未执行回收策略
* Failed：发生错误


## PV

PersistentVolume（PV）是集群之中的一块网络存储。跟 Node 一样，也是集群的资源。PV 跟 Volume (卷) 类似，不过会有独立于 Pod 的生命周期。比如一个 NFS 的 PV 可以定义为

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /tmp
    server: 172.17.0.2
```

PV 的访问模式（accessModes）有三种：

* ReadWriteOnce（RWO）：是最基本的方式，可读可写，但只支持被单个节点挂载。
* ReadOnlyMany（ROX）：可以以只读的方式被多个节点挂载。
* ReadWriteMany（RWX）：这种存储可以以读写的方式被多个节点共享。不是每一种存储都支持这三种方式，像共享方式，目前支持的还比较少，比较常用的是 NFS。在 PVC 绑定 PV 时通常根据两个条件来绑定，一个是存储的大小，另一个就是访问模式。

PV 的回收策略（persistentVolumeReclaimPolicy，即 PVC 释放卷的时候 PV 该如何操作）也有三种

* Retain，不清理, 保留 Volume（需要手动清理）
* Recycle，删除数据，即 rm -rf /thevolume/*（只有 NFS 和 HostPath 支持）
* Delete，删除存储资源，比如删除 AWS EBS 卷（只有 AWS EBS, GCE PD, Azure Disk 和 Cinder 支持）


## StorageClass

上面通过手动的方式创建了一个 NFS Volume，这在管理很多 Volume 的时候不太方便。Kubernetes 还提供了 StorageClass 来动态创建 PV，不仅节省了管理员的时间，还可以封装不同类型的存储供 PVC 选用。

StorageClass 包括四个部分

* provisioner：指定 Volume 插件的类型，包括内置插件（如 kubernetes.io/glusterfs）和外部插件（如 external-storage 提供的 ceph.com/cephfs）。
* mountOptions：指定挂载选项，当 PV 不支持指定的选项时会直接失败。比如 NFS 支持 hard 和 nfsvers=4.1 等选项。
* parameters：指定 provisioner 的选项，比如 kubernetes.io/aws-ebs 支持 type、zone、iopsPerGB 等参数。
* reclaimPolicy：指定回收策略，同 PV 的回收策略。

在使用 PVC 时，可以通过 DefaultStorageClass 准入控制设置默认 StorageClass, 即给未设置 storageClassName 的 PVC 自动添加默认的 StorageClass。而默认的 StorageClass 带有 annotation storageclass.kubernetes.io/is-default-class=true。

GCE 示例，单个 GCE 节点最大支持挂载 16 个 Google Persistent Disk。开启 AttachVolumeLimit 特性后，根据节点的类型最大可以挂载 128 个。

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  zone: us-central1-a
```

Glusterfs 示例

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://127.0.0.1:8081"
  clusterid: "630372ccdc720a92c681fb928f27b53f"
  restauthenabled: "true"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-secret"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "replicate:3"
```

OpenStack Cinder 示例

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gold
provisioner: kubernetes.io/cinder
parameters:
  type: fast
  availability: nova
```

Ceph RBD 示例

```yaml
apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: fast
  provisioner: kubernetes.io/rbd
  parameters:
    monitors: 10.16.153.105:6789
    adminId: kube
    adminSecretName: ceph-secret
    adminSecretNamespace: kube-system
    pool: kube
    userId: kube
    userSecretName: ceph-secret-user
```

Local Volume 允许将 Node 本地的磁盘、分区或者目录作为持久化存储使用。注意，Local Volume 不支持动态创建，使用前需要预先创建好 PV。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  # volumeMode field requires BlockVolume Alpha feature gate to be enabled.
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```


## PVC

PV 是存储资源，而 PersistentVolumeClaim (PVC) 是对 PV 的请求。PVC 跟 Pod 类似：Pod 消费 Node 资源，而 PVC 消费 PV 资源；Pod 能够请求 CPU 和内存资源，而 PVC 请求特定大小和访问模式的数据卷。


```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

PVC 可以直接挂载到 Pod 中：

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: dockerfile/nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```


# Pod

Pod 是一组紧密关联的容器集合，它们共享 IPC、Network 和 UTS namespace，是 Kubernetes 调度的基本单位。Pod 的设计理念是支持多个容器在一个 Pod 中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。

![pod](images/pod.png "pod")

Pod 的特征:

* 包含多个共享 IPC、Network 和 UTC namespace 的容器，可直接通过 localhost 通信
* 所有 Pod 内容器都可以访问共享的 Volume，可以访问共享数据
* 无容错性：直接创建的 Pod 一旦被调度后就跟 Node 绑定，即使 Node 挂掉也不会被重新调度（而是被自动删除），因此推荐使用 Deployment、Daemonset 等控制器来容错
* 优雅终止：Pod 删除的时候先给其内的进程发送 SIGTERM，等待一段时间（grace period）后才强制停止依然还在运行的进程
* 特权容器（通过 SecurityContext 配置）具有改变系统配置的权限（在网络插件中大量应用）


## Pod 生命周期

Kubernetes 以 PodStatus.Phase 抽象 Pod 的状态（但并不直接反映所有容器的状态）。可能的 Phase 包括:

* Pending: Pod 已经在 apiserver 中创建，但还没有调度到 Node 上面
* Running: Pod 已经调度到 Node 上面，所有容器都已经创建，并且至少有一个容器还在运行或者正在启动
* Succeeded: Pod 调度到 Node 上面后成功运行结束，并且不会重启
* Failed: Pod 调度到 Node 上面后至少有一个容器运行失败（即退出码不为 0 或者被系统终止）
* Unknonwn: 状态未知，通常是由于 apiserver 无法与 kubelet 通信导致




