# VirtualService 

VirtualService 由一组路由规则组成，用于对服务实体（在 Kubernetes 中对应为 Pod）进行寻址。如果有流量命中了某条路由规则，就会将其发送到对应的服务或者服务的一个版本/子集。

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:      # string[]，域名匹配规则，VirtualService中的规则只影响与hosts匹配的请求，可以是 IP 地址、DNS 名称，或者依赖于平台的一个简称（FQDN）。
  http:	      # HTTPRoute[]， HTTP流量规则配置
  tls:        # TLSRoute[]，TLS & HTTPS流量规则配置
  exportTo:   # string[], virtualservice export到指定的namespace，即规则在该命名空间中的sidecar和gateway生效。
  gateways:   # string[]，配置流量规则的gateway和sidecar。
```

## 分流-默认规则

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews		# 域名匹配规则，VirtualService中的规则只影响与hosts匹配的请求，可以是 IP 地址、DNS 名称，或者依赖于平台的一个简称（FQDN）。
  http:
  - route:
    - destination:	# 默认路由规则，确保流经虚拟服务的流量至少能够匹配一条路由规则
        host: reviews
```

## 分流-服务subset

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews		# 域名匹配规则，VirtualService中的规则只影响与hosts匹配的请求，可以是 IP 地址、DNS 名称，或者依赖于平台的一个简称（FQDN）。
  http:
  - match:		# 匹配规则，路由规则按从上到下的顺序选择，虚拟服务中定义的第一条规则有最高优先级。
    - headers:
        end-user:
          exact: jason	# 匹配条件， end-user header值为jason（精确匹配）
    route:
    - destination:	# 符合匹配条件的流量目标地址，destination 的 host 必须是存在于 Istio 服务注册中心的实际目标地址。
        host: reviews	# host 必须是存在于 Istio 服务注册中心的实际目标地址，否则 Envoy 不知道该将请求发送到哪里。
        subset: v2
  - route:
    - destination:	# 默认路由规则，确保流经虚拟服务的流量至少能够匹配一条路由规则
        host: reviews
        subset: v3
```

## 分流-多服务

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com			# 匹配同一个域名
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews		# /reviews前缀的请求，路由到reviews服务
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings		# /ratings前缀的请求，路由到ratings服务
```

## 分流-默认路由权重分流

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - route:                 # 默认路由规则
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
      weight: 25
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
      weight: 75
```

## 修改header

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - headers:
      request:
        set:			# 修改请求header，如果添加header使用add，HTTPRoute对象中设置
          test: true
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
      weight: 25
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
      headers:
        response:
          remove:		# 删除响应header，HTTPRouteDestination对象中设置
          - foo
      weight: 75
```

## 故障注入-延时

```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 7s  # 延时7秒
        percentage:
          value: 100    # 100%延时
    match:
    - headers:
        end-user:
          exact: jason   # 针对jason用户调用ratings服务
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings   # 其他用户不注入延时
        subset: v1
```

## 请求超时

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-productpage-rule
spec:
  hosts:
  - productpage.prod.svc.cluster.local
  http:
  - timeout: 5s			# 请求超时为，HTTPRoute对象中设置
    route:
    - destination:
        host: productpage.prod.svc.cluster.local
```

## 请求重试

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
    retries:			# 请求重试，HTTPRoute对象中设置
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
```

## 重定向

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - uri:
        exact: /v1/getProductRatings
    redirect:
      uri: /v1/bookRatings                              # 重定向地址， 默认的重定向code为301，HTTPRoute对象中设置
      authority: newratings.default.svc.cluster.local   # 修改URL中的Authority/Host值
```

## Path重写

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: /ratings
    rewrite:
      uri: /v1/bookRatings				# 重写path，HTTPRoute对象中设置
    route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
```

## 镜像

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2       # 流量镜像到 v2版本
    mirrorPercent: 100
```

# DestinationRule

## 多分组

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local	# 注册中心服务（k8s+serviceentry）
  trafficPolicy:			# 服务默认的负载均衡策略
    loadBalancer:
      simple: LEAST_CONN
  subsets:				    # 分组
  - name: testversion
    labels:
      version: v3
    trafficPolicy:			# 分组的负载均衡策略
      loadBalancer:
        simple: ROUND_ROBIN
```

## 基于端口的负载均衡策略

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings-port
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy: # Apply to all ports
    portLevelSettings:
    - port:
        number: 80
      loadBalancer:
        simple: LEAST_CONN
    - port:
        number: 9080
      loadBalancer:
        simple: ROUND_ROBIN
```

## 连接池

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-redis
spec:
  host: myredissrv.prod.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:				# TCP连接池配置
        maxConnections: 100
        connectTimeout: 30ms
        tcpKeepalive:
          time: 7200s
          interval: 75s
```

## 被动健康检查

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-cb-policy
spec:
  host: reviews.prod.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutiveErrors: 7
      interval: 5m
      baseEjectionTime: 15m
```

## TLS

```YAML
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: db-mtls
spec:
  host: mydbserver.prod.svc.cluster.local
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
	  
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: tls-foo
spec:
  host: "*.foo.com"     # 外部服务
  trafficPolicy:
    tls:
      mode: SIMPLE

apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ratings-istio-mtls
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL       #  Istio mutual TLS
```

## 熔断

```
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
      tcp:
        maxConnections: 1
    outlierDetection:
      baseEjectionTime: 3m
      consecutive5xxErrors: 1
      interval: 1s
      maxEjectionPercent: 100
```

# Gateway

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: demo-gwy
spec:
  selector:
    app: demo-gateway-controller
servers:
  - port:
       number: 443
       name: https
       protocol: HTTPS
    hosts:
       - demo.example.com
    tls:
       mode: SIMPLE
       serverCertificate: /tmp/tls.crt  # 公钥
       privateKey: /tmp/tls.key  # 私钥
```

# Sidecar

Sidecar定义了应用流量拦截的Proxy。

## 全局默认Sidecar

全局Sidecar没有设置workloadSelector值，只允许访问当前命名空间和istio-system命名空间中的service。每个namespace只能配置一个Sidecar。

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: istio-config
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
```

## 全局自定义Sidecar

全局Sidecar没有设置workloadSelector值，允许访问prod-us1、prod-apis和istio-system命名空间中的service。

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: prod-us1
spec:
  egress:
  - hosts:
    - "prod-us1/*"
    - "prod-apis/*"
    - "istio-system/*"
```

## 服务Sidecar

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: ratings
  namespace: prod-us1	# 指定namespace
spec:
  workloadSelector:
    labels:
      app: ratings		# 指定pod，这些pod属于ratings.prod-us1服务
  ingress:
  - port:
      number: 9080		# POD接受指向9080端口的请求
      protocol: HTTP
      name: somename
    defaultEndpoint: unix:///var/run/someuds.sock   # 业务容器监听的socket
  egress:
  - port:
      number: 9080
      protocol: HTTP
      name: egresshttp
    hosts:
    - "prod-us1/*"		# 只允许访问prod-us1命名空间下9080端口的HTTP协议请求
  - hosts:
    - "istio-system/*"
```

## 非IPtables拦截的Sidecar

非iptables拦截的Sidecar需要在

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: no-ip-tables
  namespace: prod-us1
spec:
  workloadSelector:
    labels:
      app: productpage
  ingress:
  - port:
      number: 9080        # binds to proxy_instance_ip:9080 ，pod可以接受该端口的请求
      protocol: HTTP
      name: somename
    defaultEndpoint: 127.0.0.1:8080   # 业务监听在8080端口
    captureMode: NONE     # not needed if metadata is set for entire proxy
  egress:
  - port:
      number: 3306
      protocol: MYSQL
      name: egressmysql
    captureMode: NONE     # not needed if metadata is set for entire proxy
    bind: 127.0.0.1       # Sidecar监听本地的3306端口
    hosts:
    - "*/mysql.foo.com"   # 代理请求到mysql.foo.com:3306。
```

# ServiceEntry

ServiceEntry用于定义服务网格外的服务，包括网格内非K8S部署的服务，以及网格外的服务。

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:  
  name: vmapp  
  namespace: powerful
spec:  
  exportTo:   # 默认情况下使用“*”，这意味着该ServiceEntry公开给每个命名空间。“.”仅将其限制为当前命名空间。
  hosts:      # 外部服务域名，VirtualService和DestinationRule中hosts映射。
  location:   # 服务位置，MESH_EXTERNAL（外部API）和MESH_INTERNAL（网格内VM部署）两种。
  ports:      # 外部服务端口
  resolution: # 服务发现机制，NONE（IP地址已经解析过，是真实的目的IP地址）、STATIC（静态设置endpoint）和DNS（使用hosts获取IP）  
  endpoints:  # 服务endpoints配置
  workloadSelector:  # 服务关联的workloadEntry，或者k8s pod。
  subjectAltNames:   # ？？？？？
```

## 单个endpoint

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:  
  name: vmapp
  namespace: powerful
spec:  
  exportTo:  
  - '*'     # 默认情况下使用“*”，这意味着该ServiceEntry公开给每个命名空间。“.”仅将其限制为当前命名空间。
  hosts:  
  - vmapp.powerful.svc.cluster.local     # 访问地址
  location: MESH_EXTERNAL  
  ports:  
  - name: http    
    number: 80                        # 服务端口
    protocol: HTTP  
  resolution: STATIC  
  endpoints:  
    - address: 172.16.16.34       #  虚拟机的地址
      labels:      
        app: vmapp
        nsf.skiff.netease.com/platform: virutalmachine    
      ports:      
        http: 19990              # 虚拟机上服务暴露的端口
```

## 多个endpoint（IP）

```YAML
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-mongocluster
spec:
  hosts:
  - mymongodb.somedomain 	
  addresses:
  - 192.192.192.192/24      # 网络地址
  ports:
  - number: 27018           # 服务端口
    name: mongodb
    protocol: MONGO
  location: MESH_INTERNAL
  resolution: STATIC
  endpoints:
  - address: 2.2.2.2
  - address: 3.3.3.3
```

## 多个endpoint（域名）

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-dns
spec:
  hosts:
  - foo.bar.com
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  endpoints:
  - address: us.foo.bar.com
    ports:
      http: 8080
  - address: uk.foo.bar.com
    ports:
      http: 9080
  - address: in.foo.bar.com
    ports:
      http: 7080
```

## 指定WorkloadEntry

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: details-svc
spec:
  hosts:
  - details.bookinfo.com
  location: MESH_INTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
    targetPort: 8080	# The port number on the endpoint where the traffic will be received.
  resolution: STATIC
  workloadSelector:
    labels:
      app: details-legacy	# 指向携带app: details-legacy标签的WorkloadEntry
```

## 外部API服务

```YAML
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-https
spec:
  hosts:
  - api.dropboxapi.com
  - www.googleapis.com
  - api.facebook.com
  location: MESH_EXTERNAL	# 网格外部
  ports:
  - number: 443
    name: https
    protocol: TLS
  resolution: DNS		# 解析hosts中的域名
```

## client本地socket服务

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: unix-domain-socket-example
spec:
  hosts:
  - "example.unix.local"
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: STATIC
  endpoints:
  - address: unix:///var/run/example/socket    # client本地socket服务
```

# WorkloadEntry

WorkloadEntry用于定义一个非K8S部署的工作负载，例如VM或物理机上部署的应用。

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadEntry
metadata:
  name: details-svc
spec:
  # use of the service account indicates that the workload has a
  # sidecar proxy bootstrapped with this service account. Pods with
  # sidecars will automatically communicate with the workload using
  # istio mutual TLS.
  serviceAccount:   # string，service account（所属namespace中必需包含该service account）
  address:          # string， 工作负载地址
  labels:           # map<string, string>， 工作负载标签
  ports:            # map<string, uint32>，工作负载端口
  network:          # string，工作负载所属网络
  locality:         # string，工作负载所属可用区
  weight:           # uint32，工作负载流量权重
```

## 静态IP地址的工作负载

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadEntry
metadata:
  name: details-svc
spec:
  # use of the service account indicates that the workload has a
  # sidecar proxy bootstrapped with this service account. Pods with
  # sidecars will automatically communicate with the workload using
  # istio mutual TLS.
  serviceAccount: details-legacy
  address: 2.2.2.2                 # 指定IP地址
  labels:
    app: details-legacy
    instance-id: vm1
```

## 域名地址的工作负载

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadEntry
metadata:
  name: details-svc
spec:
  # use of the service account indicates that the workload has a
  # sidecar proxy bootstrapped with this service account. Pods with
  # sidecars will automatically communicate with the workload using
  # istio mutual TLS.
  serviceAccount: details-legacy
  address: vm1.vpc01.corp.net       # 指定域名
  labels:
    app: details-legacy
    instance-id: vm1
```

# WorkloadGroup 

WorkloadGroup 主要用于 WorkloadEntry 自动注册。

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: reviews
  namespace: bookinfo
spec:
  metadata:  # ObjectMeta，WorkloadEntry的labels和annotations定义
  template:  # WorkloadEntry，生成WorkloadEntry的模板
  probe:     # ReadinessProbe，workload健康检查
```

## 示例

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: reviews                #  vm 上 reviews 实例启动后，都会自动在 mesh 中创建一个WorkloadEntry。
  namespace: bookinfo
spec:
  metadata:
    labels:
      app.kubernetes.io/name: reviews
      app.kubernetes.io/version: "1.3.4"
  template:
    ports:
      grpc: 3550
      http: 8080
    serviceAccount: default
  probe:
    initialDelaySeconds: 5
    timeoutSeconds: 3
    periodSeconds: 4
    successThreshold: 3
    failureThreshold: 3
    httpGet:
     path: /foo/bar
     host: 127.0.0.1
     port: 3100
     scheme: HTTPS
     httpHeaders:
     - name: Lit-Header
       value: Im-The-Best
```

# 应用场景

## egress gateway代理外部服务

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-httpbin
  namespace : egress
spec:
  hosts:
  - httpbin.com			# 外部服务域名
  exportTo:
  - "."					# 该服务仅对本命名空间开放
  location: MESH_EXTERNAL	# 外部API服务
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS		# 通过DNS解析域名
```

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
 name: istio-egressgateway   # 网关名
 namespace: istio-system
spec:
 selector:
   istio: egressgateway		# 选择网关POD
 servers:
 - port:
     number: 80
     name: http
     protocol: HTTP
   hosts:
   - "*"					# 所有域名
```

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: gateway-routing
  namespace: egress
spec:
  hosts:
  - httpbin.com
  exportTo:
  - "*"
  gateways:
  - mesh
  - istio-egressgateway
  http:
  - match:
    - port: 80
      gateways:         # ？？？？？？？？
      - mesh
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local     # 转发请求到gateway
  - match:
    - port: 80
      gateways:
      - istio-egressgateway
    route:
    - destination:
        host: httpbin.com        # 转发请求道httpbin.com服务，即ServiceEntry定义的外部服务
```



