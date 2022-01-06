# 外部访问

提供集群内跨容器访问，以及集群外访问容器的能力，提供如下资源：

* service
* ingress


## Service 

Service 四种类型:

* ClusterIP：默认类型，自动分配一个仅 cluster 内部可以访问的虚拟 IP
* NodePort：在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口，这样就可以通过 <NodeIP>:NodePort 来访问该服务。
* LoadBalancer：在 NodePort 的基础上，借助 cloud provider 创建一个外部的负载均衡器，并将请求转发到 <NodeIP>:NodePort
* ExternalName：将服务通过 DNS CNAME 记录方式转发到指定的域名（通过 spec.externlName 设定）。需要 kube-dns 版本在 1.7 以上。

service资源定义：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx			# service的标签
  name: nginx			# service 名称
  namespace: default    # 服务所属命名空间，决定了其访问方式
spec:
  ports:
  - port: 30080			# service暴露在Cluster上的端口，即在集群内采用 ClusterIP:Port 访问容器
    protocol: TCP		# 协议类型
    targetPort: 80		# 容器的端口，也是最终底层的服务所提供的端口，所以说targetPod也就是Pod的端口
	nodePort:30003		# 外部流量访问K8s的一种方式，即nodeIP:nodePort
  selector:
    run: nginx			# service关联的pod资源
  sessionAffinity: None # service负载均衡，默认值是None，根据iptables规则随机调度；可使用sessionAffinity保持会话连线；
  type: ClusterIP		# service类型
```


### ClusterIP服务

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


### NodePort服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  selector:
      app: nginx
  ports:
    - name: http
      port: 8000
      protocol: TCP
      targetPort: 80
    - name: https
      port: 8443
      protocol: TCP
      targetPort: 443
  type: NodePort			## 类型为NodePort
```


### LoadBalancer服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
spec:
  selector:
    app: example
  ports:
    - port: 8765
      targetPort: 9376
  externalTrafficPolicy: Local	# 表示此服务是否希望将外部流量路由到节点本地或集群范围的端点。有两个可用选项：Cluster（默认）和 Local。
  type: LoadBalancer			# 类型为LoadBalancer
```

externalTrafficPolicy类型说明：

* Cluster 隐藏了客户端源 IP，可能导致第二跳到另一个节点，但具有良好的整体负载分布。
* Local 保留客户端源 IP 并避免 LoadBalancer 和 NodePort 类型服务的第二跳，但存在潜在的不均衡流量传播风险。


### externalName服务

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


### Headless服务

通过 <service-name>.<namespace>.svc.cluster.local 访问服务，DNS请求的结果实际为Pod IP地址。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  clusterIP: None		# 不分配Cluster IP
  ports:
  - name: tcp-80-80-3b6tl
    port: 80
    protocol: TCP
    targetPort: 80
  selector:				# selector方式
    app: nginx
  sessionAffinity: None
  type: ClusterIP		# 类型还是Cluster IP类型
```


### 暴露endpoint

创建同名的 service 和 endpoint，在 endpoint 中设置外部服务的 IP 和端口。

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


## Ingress

Ingress 是 k8s 资源对象，用于对外暴露服务，该资源对象定义了不同主机名（域名）及 URL 和对应后端 Service（k8s Service）的绑定，根据不同的路径路由 http 和 https 流量。

Ingress资源定义:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: "nginx"					# Nginx Ingress Controller 根据该注解自动发现 Ingress
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"	# Controller 向后端 Service 转发时使用 HTTPS 协议，这个注解必须添加，否则访问会报错
spec:
  tls:
    - secretName: testsecret					# https 证书 Secret
  backend:
    serviceName: s1								# 默认暴露的service 名字
    servicePort: 80								# 默认暴露的service 端口
  rules:
  - host: foo.bar.com							# 域名匹配
    http:
      paths:
      - path: /testpath							# 匹配访问路径
        backend:
          serviceName: test						# 对外暴露的 Service 名称
          servicePort: 80						# 对外暴露的 service 端口
```


### 单服务Ingress 

该 Ingress 仅指定一个没有任何规则的后端服务。

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


### 多服务Ingress

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


### 虚拟主机Ingress 

根据域名的不同转发到不同的后端服务上，而他们共用同一个的 IP 地址。

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


### TLS Ingress 

Secret 获取 TLS 私钥和证书 (名为 tls.crt 和 tls.key)，来执行 TLS 终止。

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


# 容器


## Pod

Pod 是一组紧密关联的容器集合，它们共享 IPC、Network 和 UTS namespace，是 Kubernetes 调度的基本单位。Pod 的设计理念是支持多个容器在一个 Pod 中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。

**Pod 的特征**

* 包含多个共享 IPC、Network 和 UTC namespace 的容器，可直接通过 localhost 通信
* 所有 Pod 内容器都可以访问共享的 Volume，可以访问共享数据
* 无容错性：直接创建的 Pod 一旦被调度后就跟 Node 绑定，即使 Node 挂掉也不会被重新调度（而是被自动删除），因此推荐使用 Deployment、Daemonset 等控制器来容错
* 优雅终止：Pod 删除的时候先给其内的进程发送 SIGTERM，等待一段时间（grace period）后才强制停止依然还在运行的进程
* 特权容器（通过 SecurityContext 配置）具有改变系统配置的权限（在网络插件中大量应用）


**Pod 生命周期**

* Pending: Pod 已经在 apiserver 中创建，但还没有调度到 Node 上面
* Running: Pod 已经调度到 Node 上面，所有容器都已经创建，并且至少有一个容器还在运行或者正在启动
* Succeeded: Pod 调度到 Node 上面后成功运行结束，并且不会重启
* Failed: Pod 调度到 Node 上面后至少有一个容器运行失败（即退出码不为 0 或者被系统终止）
* Unknonwn: 状态未知，通常是由于 apiserver 无法与 kubelet 通信导致

**Pod资源定义：**

```yaml
apiVersion: v1        　　          #必选，版本号，例如v1,版本号必须可以用 kubectl api-versions 查询到 .
kind: Pod       　　　　　　         #必选，Pod
metadata:       　　　　　　         #必选，元数据
  name: string        　　          #必选，Pod名称
  namespace: string     　　        #必选，Pod所属的命名空间,默认为"default"
  labels:       　　　　　　          #自定义标签
    - name: string      　          #自定义标签名字
  annotations:        　　                 #自定义注释列表
    - name: string
spec:         　　　　　　　            #必选，Pod中容器的详细定义
  containers:       　　　　            #必选，Pod中容器列表
  - name: string      　　                #必选，容器名称,需符合RFC 1035规范
    image: string     　　                #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像,否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]     　　        #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      　　             #容器的启动命令参数列表
    workingDir: string                     #容器的工作目录
    volumeMounts:     　　　　        #挂载到容器内部的存储卷配置
    - name: string      　　　        #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string                 #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean                 #是否为只读模式
    ports:        　　　　　　        #需要暴露的端口库号列表
    - name: string      　　　        #端口的名称
      containerPort: int                #容器需要监听的端口号
      hostPort: int     　　             #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string                  #端口协议，支持TCP和UDP，默认TCP
    env:        　　　　　　            #容器运行前需设置的环境变量列表
    - name: string      　　            #环境变量名称
      value: string     　　            #环境变量的值
    resources:        　　                #资源限制和请求的设置
      limits:       　　　　            #资源限制的设置
        cpu: string     　　            #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string                  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:       　　                #资源请求的设置
        cpu: string     　　            #Cpu请求，容器启动的初始可用数量
        memory: string                    #内存请求,容器启动的初始可用数量
    livenessProbe:      　　            #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:       　　　　　　        #对Pod容器内检查方式设置为exec方式
        command: [string]               #exec方式需要制定的命令或脚本
      httpGet:        　　　　        #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:      　　　　　　#对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
    restartPolicy: [Always | Never | OnFailure] #Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject   　　    #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:     　　　　#Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork: false      　　    #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:        　　　　　　    #在该pod上定义共享存储卷列表
    - name: string     　　 　　    #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}      　　　　    #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string      　　    #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:       　　　　　　    #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:      　　　　            #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
```


### 使用PVC挂载块设备

```yaml
# Persistent Volumes using a Raw Block Volume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block			# 块设备
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
---
# Persistent Volume Claim requesting a Raw Block Volume
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block		# 指定块设备
  resources:
    requests:
      storage: 10Gi		# 申请10G大小
---
# Pod specification adding Raw Block Device path in container
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
  annotations:
    # apparmor should be unconfied for mounting the device inside container.
    container.apparmor.security.beta.kubernetes.io/fc-container: unconfined
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: ["tail -f /dev/null"]
      securityContext:
        capabilities:
          # CAP_SYS_ADMIN is required for mount() syscall.
          add: ["SYS_ADMIN"]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda		# 块设备挂载的路径
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc		# 指定PVC
```


### 挂载hostPath卷

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod2
spec:
  containers:
  - image: busybox
    name: test-hostpath
    command: [ "sleep", "3600" ]
    volumeMounts:
    - mountPath: /test-data
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```

## Deployment

Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义 (declarative) 方法，用来替代以前的 ReplicationController 来方便的管理应用。

**Deployment定义：**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment		# deployment名称
  labels:
    app: nginx				# deployment的标签
spec:
  replicas: 3				# 副本数
  selector:				
    matchLabels:			# deployment管理的pod
      app: nginx
  minReadySeconds: 5			# 表示 Kubernetes 在等待设置的时间后才进行升级，如果没有设置该值，Kubernetes 会假设该容器启动起来后就提供服务了
  strategy:  				# 滚动更新策略
    type: RollingUpdate  		# 设置更新策略为滚动更新，可以设置为Recreate和RollingUpdate两个值，Recreate表示全部重新创建，默认值就是RollingUpdate
    rollingUpdate:
      maxSurge: 1			# 表示升级过程中最多可以比原先设置多出的 Pod 数量
      maxUnavailable: 1			# 表示升级过程中最多有多少个 Pod 处于无法提供服务的状态
  template:				# pod模板，它与pod的样式完全相同，除了它是嵌套的，并且没有 apiVersion 和 kind。
    metadata:
      labels:
        app: nginx			# 创建的pod会在.metadata.labels中，添加app: nginx标签
    spec:
      containers:
      - name: nginx			# 容器名称
        image: nginx:1.7.9		# 容器镜像
        ports:
        - containerPort: 80		# 容器暴露的端口
```


### 多端口

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name:  nginx
        image: nginx
        ports:
        - containerPort: 80
        - containerPort: 443
```


## StatefulSet

StatefulSet 是为了解决有状态服务的问题（对应 Deployments 和 ReplicaSets 是为无状态服务而设计），其应用场景包括：

* 稳定的持久化存储，即 Pod 重新调度后还是能访问到相同的持久化数据，基于 PVC 来实现
* 稳定的网络标志，即 Pod 重新调度后其 PodName 和 HostName 不变，基于 Headless Service（即没有 Cluster IP 的 Service）来实现
* 有序部署，有序扩展，即 Pod 是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依序进行（即从 0 到 N-1，在下一个 Pod 运行之前所有之前的 Pod 必须都是 Running 和 Ready 状态），基于 init containers 来实现
* 有序收缩，有序删除（即从 N-1 到 0）

**StatefulSet组成：**

* 用于定义网络标志（DNS domain）的 Headless Service
* 用于创建 PersistentVolumes 的 volumeClaimTemplates
* 定义具体应用的 StatefulSet
* StatefulSet 中每个 Pod 的 DNS 格式为 statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local，其中
  * serviceName 为 Headless Service 的名字
  * 0..N-1 为 Pod 所在的序号，从 0 开始到 N-1
  * statefulSetName 为 StatefulSet 的名字
  * namespace 为服务所在的 namespace，Headless Service 和 StatefulSet 必须在相同的 namespace
  * .cluster.local 为 Cluster Domain

StatefulSet 需要一个 Headless Service 来定义 DNS domain，需要在 StatefulSet 之前创建好


### StatefulSet示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:						# 
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi

```

```cmd
$ kubectl create -f web.yaml
service "nginx" created
statefulset "web" created

# 查看创建的 headless service 和 statefulset
$ kubectl get service nginx
NAME      CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     None         <none>        80/TCP    1m

$ kubectl get statefulset web
NAME      DESIRED   CURRENT   AGE
web       2         2         2m

# 根据 volumeClaimTemplates 自动创建 PVC（在 GCE 中会自动创建 kubernetes.io/gce-pd 类型的 volume）
$ kubectl get pvc
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-d064a004-d8d4-11e6-b521-42010a800002   1Gi        RWO           16s
www-web-1   Bound     pvc-d06a3946-d8d4-11e6-b521-42010a800002   1Gi        RWO           16s

# 查看创建的 Pod，他们都是有序的
$ kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          5m
web-1     1/1       Running   0          4m
```


## DaemonSet

DaemonSet 保证在每个 Node 上都运行一个容器副本，常用来部署一些集群的日志、监控或者其他系统管理应用。典型的应用包括：

* 日志收集，比如 fluentd，logstash 等
* 系统监控，比如 Prometheus Node Exporter，collectd，New Relic agent，Ganglia gmond 等
* 系统程序，比如 kube-proxy, kube-dns, glusterd, ceph 等


### DaemonSet示例

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: gcr.io/google-containers/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```


## Autoscaling 

Horizontal Pod Autoscaling (HPA) 可以根据 CPU 使用率或应用自定义 metrics 自动扩展 Pod 数量（支持 replication controller、deployment 和 replica set ）。

* 控制管理器每隔 30s（可以通过 --horizontal-pod-autoscaler-sync-period 修改）查询 metrics 的资源使用情况
* 支持三种 metrics 类型
  * 预定义 metrics（比如 Pod 的 CPU）以利用率的方式计算
  * 自定义的 Pod metrics，以原始值（raw value）的方式计算
  * 自定义的 object metrics
* 支持两种 metrics 查询方式：Heapster 和自定义的 REST API
* 支持多 metrics


### Autoscaling示例

比如 HorizontalPodAutoscaler 保证每个 Pod 占用 50% CPU、1000pps 以及 10000 请求 / s

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
  - type: Pods
    pods:
      metricName: packets-per-second
      targetAverageValue: 1k
  - type: Object
    object:
      metricName: requests-per-second
      target:
        apiVersion: extensions/v1beta1
        kind: Ingress
        name: main-route
      targetValue: 10k
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
      currentAverageUtilization: 0
      currentAverageValue: 0
```


# 配置

**Secret 与 ConfigMap 相同点：**

* key/value 的形式
* 属于某个特定的 namespace
* 可以导出到环境变量
* 可以通过目录 / 文件形式挂载 (支持挂载所有 key 和部分 key)


**Secret 与 ConfigMap 不同点：**

* Secret 可以被 ServerAccount 关联 (使用)
* Secret 可以存储 register 的鉴权信息，用在 ImagePullSecret 参数中，用于拉取私有仓库的镜像
* Secret 支持 Base64 加密
* Secret 分为 Opaque，kubernetes.io/Service Account，kubernetes.io/dockerconfigjson 三种类型, Configmap 不区分类型
* Secret 文件存储在 tmpfs 文件系统中，Pod 删除后 Secret 文件也会对应的删除。


## Secret

Secret 解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者 Pod Spec 中。Secret 可以以 Volume 或者环境变量的方式使用。

**Secret 有三种类型：**

* Opaque：base64 编码格式的 Secret，用来存储密码、密钥等；但数据也通过 base64 --decode 解码得到原始数据，所有加密性很弱。
* kubernetes.io/dockerconfigjson：用来存储私有 docker registry 的认证信息。
* kubernetes.io/service-account-token： 用于被 serviceaccount 引用。serviceaccout 创建时 Kubernetes 会默认创建对应的 secret。Pod 如果使用了 serviceaccount，对应的 secret 会自动挂载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中。
  * serviceaccount 用来使得 Pod 能够访问 Kubernetes API


### Opaque Secret

```cmd
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
```

将 Secret 挂载到 Volume 中

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: db
  name: db
spec:
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
  containers:
  - image: gcr.io/my_project_id/pg:v1
    name: db
    volumeMounts:
    - name: secrets
      mountPath: "/etc/secrets"
      readOnly: true
    ports:
    - name: cp
      containerPort: 5432
      hostPort: 5432
```

查看 Pod 中对应的信息：

```cmd
# ls /etc/secrets
password  username
# cat  /etc/secrets/username
admin
# cat  /etc/secrets/password
1f2d1e2e67df
```

将 Secret 导出到环境变量中

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: wordpress-deployment
spec:
  replicas: 2
  strategy:
      type: RollingUpdate
  template:
    metadata:
      labels:
        app: wordpress
        visualize: "true"
    spec:
      containers:
      - name: "wordpress"
        image: "wordpress"
        ports:
        - containerPort: 80
        env:							# 环境变量
        - name: WORDPRESS_DB_USER		# key
          valueFrom:
            secretKeyRef:
              name: mysecret			# 对应的secret
              key: username				# secret的key
        - name: WORDPRESS_DB_PASSWORD	# key
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
```

将 Secret 挂载指定的 key

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: db
  name: db
spec:
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
      items:
      - key: password		# secret key
        mode: 511
        path: tst/psd		# password 映射到路径
      - key: username
        mode: 511
        path: tst/usr
  containers:
  - image: nginx
    name: db
    volumeMounts:
    - name: secrets
      mountPath: "/etc/secrets"
      readOnly: true
    ports:
    - name: cp
      containerPort: 80
      hostPort: 5432
```

创建 Pod 成功后，可以在对应的目录看到：

```cmd
# kubectl exec db ls /etc/secrets/tst
psd
usr
```


### kubernetes.io/service-account-token

kubernetes.io/service-account-token： 用于被 serviceaccount 引用。serviceaccout 创建时 Kubernetes 会默认创建对应的 secret。Pod 如果使用了 serviceaccount，对应的 secret 会自动挂载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中。



## ConfigMap

在执行应用程式或是生产环境等等, 会有许多的情况需要做变更, 而我们不希望因应每一种需求就要准备一个镜像档, 这时就可以透过 ConfigMap 来帮我们做一个配置档或是命令参数的映射, 更加弹性化使用我们的服务或是应用程式。

ConfigMap 用于保存配置数据的键值对，可以用来保存单个属性，也可以用来保存配置文件。ConfigMap 跟 secret 很类似，但它可以更方便地处理不包含敏感信息的字符串。


### 从 key-value 字符串创建

```cmd
$ kubectl create configmap special-config --from-literal=special.how=very
configmap "special-config" created
$ kubectl get configmap special-config -o go-template='{{.data}}'
map[special.how:very]
```


### 从 env 文件创建

```cmd
$ echo -e "a=b\nc=d" | tee config.env
a=b
c=d
$ kubectl create configmap special-config --from-env-file=config.env
configmap "special-config" created
$ kubectl get configmap special-config -o go-template='{{.data}}'
map[a:b c:d]
```


### 从目录创建

```cmd
$ mkdir config
$ echo a>config/a
$ echo b>config/b
$ kubectl create configmap special-config --from-file=config/
configmap "special-config" created
$ kubectl get configmap special-config -o go-template='{{.data}}'
map[a:a
 b:b
]
```


### 从文件 Yaml/Json 文件创建

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```


### ConfigMap 使用

ConfigMap 可以通过三种方式在 Pod 中使用，三种分别方式为：设置环境变量、设置容器命令行参数以及在 Volume 中直接挂载文件或目录。


#### 用作环境变量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh", "-c", "env"]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
      envFrom:
        - configMapRef:
            name: env-config
  restartPolicy: Never
```

当 Pod 结束后会输出

```cmd
SPECIAL_LEVEL_KEY=very
SPECIAL_TYPE_KEY=charm
log_level=INFO
```


#### 用作命令行参数

将 ConfigMap 用作命令行参数时，需要先把 ConfigMap 的数据保存在环境变量中，然后通过 $(VAR_NAME) 的方式引用环境变量.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
  restartPolicy: Never
```

当 Pod 结束后会输出

```cmd
very charm
```


#### 文件或目录直接挂载

将创建的 ConfigMap 直接挂载至 Pod 的 /etc/config 目录下，其中每一个 key-value 键值对都会生成一个文件，key 为文件名，value 为内容

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vol-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh", "-c", "cat /etc/config/special.how"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
  restartPolicy: Never
```

当 Pod 结束后会输出

```cmd
very
```

将创建的 ConfigMap 中 special.how 这个 key 挂载到 /etc/config 目录下的一个相对路径 /keys/special.level。如果存在同名文件，直接覆盖。其他的 key 不挂载

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh","-c","cat /etc/config/keys/special.level"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.how
          path: keys/special.level
  restartPolicy: Never
```

当 Pod 结束后会输出

```cmd
very
```

ConfigMap 支持同一个目录下挂载多个 key 和多个目录。例如下面将 special.how 和 special.type 通过挂载到 /etc/config 下。并且还将 special.how 同时挂载到 /etc/config2 下。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh","-c","sleep 36000"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
      - name: config-volume2
        mountPath: /etc/config2
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.how
          path: keys/special.level
        - key: special.type
          path: keys/special.type
    - name: config-volume2
      configMap:
        name: special-config
        items:
        - key: special.how
          path: keys/special.level
  restartPolicy: Never
```

```cmd
# ls  /etc/config/keys/
special.level  special.type
# ls  /etc/config2/keys/
special.level
# cat  /etc/config/keys/special.level
very
# cat  /etc/config/keys/special.type
charm
```


#### 单独文件挂载到目录

在一般情况下 configmap 挂载文件时，会先覆盖掉挂载目录，然后再将 congfigmap 中的内容作为文件挂载进行。如果想不对原来的文件夹下的文件造成覆盖，只是将 configmap 中的每个 key，按照文件的方式挂载到目录下，可以使用 subpath 参数。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: nginx
      command: ["/bin/sh","-c","sleep 36000"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/nginx/special.how
        subPath: special.how
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.how
          path: special.how
  restartPolicy: Never
```

```cmd
root@dapi-test-pod:/# ls /etc/nginx/
conf.d    fastcgi_params    koi-utf  koi-win  mime.types  modules  nginx.conf  scgi_params    special.how  uwsgi_params  win-utf
root@dapi-test-pod:/# cat /etc/nginx/special.how
very
root@dapi-test-pod:/#
```


# 网络策略

随着微服务的流行，越来越多的云服务平台需要大量模块之间的网络调用。Kubernetes 在 1.3 引入了 Network Policy，Network Policy 提供了基于策略的网络控制，用于隔离应用并减少攻击面。它使用标签选择器模拟传统的分段网络，并通过策略控制它们之间的流量以及来自外部的流量。


## NetworkPolicy


### 默认容器间 Egress策略

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


### 默认容器间 Ingress策略

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


### 默认容器间 Egress&Ingress策略

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


### Pod 隔离策略

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


# 存储

Kubernetes 按时间顺序先后提供了 emptyDir、hostPath 和 local volume 存储卷解决方案。

**emptyDir**

emptyDir 类型的 Volume 在 Pod 分配到 Node 上时被创建，Kubernetes 会在 Node 上自动分配一个目录，因此无需指定宿主机 Node 上对应的目录文件。 这个目录的初始内容为空，当 Pod 从 Node 上移除时，emptyDir 中的数据会被永久删除。

emptyDir使用实践：

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - image: busybox
    name: test-emptydir
    command: [ "sleep", "3600" ]
    volumeMounts:
    - mountPath: /data		# 存储卷挂载到容器的目录
      name: data-volume		# 挂载的存储卷
  volumes:
  - name: data-volume
    emptyDir: {}			# 指定存储卷类型
```

**hostPath**

hostPath 类型则是映射 node 文件系统中的文件或者目录到 pod 里。在使用 hostPath 类型的存储卷时，也可以设置 type 字段，支持的类型有**文件、目录、File、Socket、CharDevice 和 BlockDevice**。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod2
spec:
  containers:
  - image: busybox
    name: test-hostpath
    command: [ "sleep", "3600" ]
    volumeMounts:
    - mountPath: /test-data
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:							# 存储卷类型
      # directory location on host
      path: /data						# 节点上存储卷的路径
      # this field is optional
      type: Directory					# 节点上存储卷的类型
```

**emptyDir和hostPath对比**

* 二者都是node节点的本地存储卷方式；
* emptyDir可以选择把数据存到tmpfs类型的本地文件系统中去，hostPath并不支持这一点；
* hostPath除了支持挂载目录外，还支持File、Socket、CharDevice和BlockDevice，既支持把已有的文件和目录挂载到容器中，也提供了“如果文件或目录不存在，就创建一个”的功能；
* emptyDir是临时存储空间，完全不提供持久化支持；
* hostPath的卷数据是持久化在node节点的文件系统中的，即便pod已经被删除了，volume卷中的数据还会留存在node节点上；

**localVolume**

本地数据卷（Local Volume）代表一个本地存储设备，比如磁盘、分区或者目录等。主要的应用场景包括分布式存储和数据库等需要高性能和高可靠性的环境里。本地数据卷同时支持块设备和文件系统，通过 spec.local.path 指定；但对于文件系统来说，kubernetes 并不会限制该目录可以使用的存储空间大小。

本地数据卷只能以静态创建的 PV 使用。相对于 HostPath，本地数据卷可以直接以持久化的方式使用（它总是通过 NodeAffinity 调度在某个指定的节点上）。

**hostPath与local volume**

* 二者都基于node节点本地存储资源实现了容器内数据的持久化功能，都为某些特殊场景下提供了更为适用的存储解决方案；
* 前者时间很久了，所以功能稳定，而后者因为年轻，所以功能的可靠性与稳定性还需要经历时间和案例的历练，尤其是对Block设备的支持还只是alpha版本；
* 二者都为k8s存储管理提供了PV、PVC和StorageClass的方法实现；
* local volume实现的StorageClass不具备完整功能，目前只支持卷的延迟绑定；
* hostPath是单节点的本地存储卷方案，不提供任何基于node节点亲和性的pod调度管理支持；
* local volume适用于小规模的、多节点的k8s开发或测试环境，尤其是在不具备一套安全、可靠且性能有保证的存储集群服务时；


## PersistentVolume

PersistentVolume（PV）用于为用户和管理员提供如何提供和消费存储的API，PV由管理员在集群中提供的存储。它就像Node一样是集群中的一种资源。PersistentVolume 也是和存储卷一样的一种插件，但其有着自己独立的生命周期。PersistentVolumeClaim (PVC)是用户对存储的请求，类似于Pod消费Node资源，PVC消费PV资源。Pod能够请求特定的资源(CPU和内存)，声明请求特定的存储大小和访问模式。PV是一个系统的资源，因此没有所属的命名空间。

**为什么又要引入 PV 呢？**

我们知道 pod 中声明的 volume 生命周期与 pod 是相同的，以下有几种常见的场景：

1. pod 销毁重建（如Deployment管理的Pod镜像升级）
2. 宿主机故障迁移（如StatefulSet管理的Pod带远程volume迁移）
3. 多个 pod共享一个数据volume
4. 数据volume snapshot，resize等功能的扩展实现

以上场景中，通过 Pod Volumes 很难准确地表达它的复用/共享语义，对它的扩展也比较困难。因此 K8s 中又引入了 Persistent Volumes 概念，它可以将存储和计算分离，通过不同的组件来管理存储资源和计算资源，然后解耦 pod 和 Volume 之间生命周期的关联。这样，当把 pod 删除之后，它使用的 PV 仍然存在，还可以被新建的 pod 复用。

**PV类型**

* GCEPersistentDisk 
* AWSElasticBlockStore 
* AzureFile 
* AzureDisk 
* FC (Fibre Channel) 
* Flocker 
* NFS 
* iSCSI 
* RBD (Ceph Block Device) 
* CephFS 
* Cinder (OpenStack block storage) 
* Glusterfs 
* VsphereVolume 
* Quobyte Volumes 
* HostPath (多节点集群中不能工作) 
* VMware Photon 
* Portworx Volumes 
* ScaleIO Volumes

**访问模式**

* ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
* ReadOnlyMany（ROX）：只读权限，可以被多个节点挂载
* ReadWriteMany（RWX）：读写权限，可以被多个节点挂载


**回收策略**

* Retain：手动回收 
* Recycle：需要擦除后才能再使用 
* Delete：相关联的存储资产，如AWS EBS，GCE PD，Azure Disk，or OpenStack Cinder卷都会被删除

**Volume生命周期**

Volume 的生命周期包括 5 个阶段

* Provisioning，即 PV 的创建，可以直接创建 PV（静态方式），也可以使用 StorageClass 动态创建
* Binding，将 PV 分配给 PVC
* Using，Pod 通过 PVC 使用该 Volume，并可以通过准入控制 StorageObjectInUseProtection（1.9 及以前版本为 PVCProtection）阻止删除正在使用的 PVC
* Releasing，Pod 释放 Volume 并删除 PVC
* Reclaiming，回收 PV，可以保留 PV 以便下次使用，也可以直接从云存储中删除
* Deleting，删除 PV 并从云存储中删除后段存储

**volume卷状态**

* Available：空闲的资源，未绑定给PVC 
* Bound：绑定给了某个PVC 
* Released：PVC已经删除了，但是PV还没有被集群回收 
* Failed：PV在自动回收中失败了 


### NFS PV

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


### localVolume PV

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


### hostPath PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath
  labels:
    type: local
spec:
  storageClassName: manual		# 选择指定的StorageClass
  capacity:
    storage: 10Gi			# 一个 PV 对象都要指定一个存储能力，通过 PV 的 capacity 属性来设置的，目前只支持存储空间的设置，
  accessModes:
    - ReadWriteOnce			# 访问模式
  hostPath:
    path: "/data/k8s/test/hostpath"	# 该卷位于集群节点上的 /data/k8s/test/hostpath 目录
```


### gcePersistentDisk PV

```yaml
apiVersion: "v1"
kind: "PersistentVolume"
metadata:
  name: gce-disk-1
  annotations:
    volume.beta.kubernetes.io/mount-options: "discard"
spec:
  capacity:
    storage: "10Gi"
  accessModes:
    - "ReadWriteOnce"
  gcePersistentDisk:
    fsType: "ext4"
    pdName: "gce-disk-1
```


### Raw Block Volume PV

Kubernetes v1.9 新增了 Alpha 版的 Raw Block Volume，可通过设置 volumeMode: Block（可选项为 Filesystem 和 Block）来使用块存储。

支持块存储的 PV 插件包括

* Local Volume
* fc
* iSCSI
* Ceph RBD
* AWS EBS
* GCE PD
* AzureDisk
* Cinder

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block							# 指定为row block volume
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
```


## PersistentVolumeClaim

PV 是存储资源，而 PersistentVolumeClaim (PVC) 是对 PV 的请求。PVC 跟 Pod 类似：Pod 消费 Node 资源，而 PVC 消费 PV 资源；Pod 能够请求 CPU 和内存资源，而 PVC 请求特定大小和访问模式的数据卷。

可以将 PVC 狭义的理解为一个对提前创建好的 PV 的筛选策略，找到与 PVC 匹配的 PV 后，二者会进行绑定。Pod 将 PVC 当作卷来使用，并藉此访问存储资源。PVC 必须位于使用它的 Pod 所在的同一命名空间内。 集群在 Pod 的命名空间中查找 PVC，并使用它来获得绑定的 PV。 之后，卷会被挂载到宿主上并挂载到 Pod 中。

**storageClassName属性:**

* 如果storageClassName 留空（nil），集群会使用默认 StorageClass 为用户自动供应一个存储卷。很多集群环境都配置了默认的 StorageClass，或者管理员也可以自行创建默认的 StorageClass。
* 如果storageClassName 属性值设置为 ""，则被视为要请求的是没有设置存储类的 PV 卷（未设置 persistentVolume.storageClassName 注解或者注解值为 "" 的 PV，此类 PV 对象在系统中不会被删除，因为这样做可能会引起数据丢失）。
* 如果storageClassName 属性值设置为 "xxx"，即那些包含了和PVC相同storageClassName的PV，才能与PVC绑定。


### Raw Block Volume PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block		# 指定为row block volume
  resources:
    requests:
      storage: 10Gi
```


### StorageClass动态供应 PVC

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


## StorageClass

通过PV创建 Volume，这在管理很多 Volume 的时候不太方便。Kubernetes 还提供了 StorageClass 来动态创建 PV，不仅节省了管理员的时间，还可以封装不同类型的存储供 PVC 选用。

StorageClass 包括四个部分：

* provisioner：指定 Volume 插件的类型，包括内置插件（如 kubernetes.io/glusterfs）和外部插件（如 external-storage 提供的 ceph.com/cephfs）。
* mountOptions：指定挂载选项，当 PV 不支持指定的选项时会直接失败。比如 NFS 支持 hard 和 nfsvers=4.1 等选项。
* parameters：指定 provisioner 的选项，比如 kubernetes.io/aws-ebs 支持 type、zone、iopsPerGB 等参数。
* reclaimPolicy：指定回收策略，同 PV 的回收策略。

在使用 PVC 时，可以通过 DefaultStorageClass 准入控制设置默认 StorageClass, 即给未设置 storageClassName 的 PVC 自动添加默认的 StorageClass。而默认的 StorageClass 带有 annotation storageclass.kubernetes.io/is-default-class=true。

**StorageClass 定义**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard		# 用户使用这个 name 来请求指定的 SC
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain   # 回收策略
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```


### GCE类型

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


### Glusterfs类型

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


### OpenStack Cinder类型

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


### Ceph RBD 类型

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


# 平台


## CronJob

CronJob 即定时任务，就类似于 Linux 系统的 crontab，在指定的时间周期运行指定的任务。

**CronJob 定义：**

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"		# 指定任务运行周期，格式同 Cron
  startingDeadlineSeconds:		# 指定任务开始的截止期限
  concurrencyPolicy:			# 指定任务的并发策略，支持 Allow、Forbid 和 Replace 三个选项
  jobTemplate:					# 指定需要运行的任务，格式同 Job
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```


## Job

Job 负责批量处理短暂的一次性任务 (short lived one-off tasks)，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。

Kubernetes 支持以下几种 Job：

* 非并行 Job：通常创建一个 Pod 直至其成功结束
* 固定结束次数的 Job：设置 .spec.completions，创建多个 Pod，直到 .spec.completions 个 Pod 成功结束
* 带有工作队列的并行 Job：设置 .spec.Parallelism 但不设置 .spec.completions，当所有 Pod 结束并且至少一个成功时，Job 就认为是成功

**Job 定义：**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    metadata:
      name: pi
    spec:
	  completions:2   # 标志 Job 结束需要成功运行的 Pod 个数，默认为 1
	  parallelism:2   # 标志并行运行的 Pod 的个数，默认为 1
	  activeDeadlineSeconds: 30  # 标志失败 Pod 的重试最大时间，超过这个时间不会继续重试
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```


### Job示例

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: busybox
spec:
  completions: 3
  template:
    metadata:
      name: busybox
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["echo", "hello"]
      restartPolicy: Never
```


## Namespace

Namespace 是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。常见的 pod, service, replication controller 和 deployment 等都是属于某一个 namespace 的（默认是 default），而 node, persistent volume，namespace 等资源则不属于任何 namespace。

Namespace 常用来隔离不同的用户，比如 Kubernetes 自带的服务一般运行在 kube-system namespace 中。


### 创建 Namespace

```cmd
(1) 命令行直接创建
$ kubectl create namespace new-namespace

(2) 通过文件创建
$ cat my-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: new-namespace

$ kubectl create -f ./my-namespace.yaml
```


## Node

Node 是 Pod 真正运行的主机，可以是物理机，也可以是虚拟机。为了管理 Pod，每个 Node 节点上至少要运行 container runtime（比如 docker 或者 rkt）、kubelet 和 kube-proxy 服务。


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

