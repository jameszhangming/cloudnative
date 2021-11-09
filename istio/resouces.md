# VirtualService 

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews		# 服务地址，可以是 IP 地址、DNS 名称，或者依赖于平台的一个简称。
  http:
  - match:		# 匹配规则，路由规则按从上到下的顺序选择，虚拟服务中定义的第一条规则有最高优先级。
    - headers:
        end-user:
          exact: jason	# 匹配条件， end-user header值为jason
    route:
    - destination:	# 符合匹配条件的流量目标地址，destination 的 host 必须是存在于 Istio 服务注册中心的实际目标地址。
        host: reviews	# host 必须是存在于 Istio 服务注册中心的实际目标地址，否则 Envoy 不知道该将请求发送到哪里。
        subset: v2
  - route:
    - destination:	# 默认路由规则，确保流经虚拟服务的流量至少能够匹配一条路由规则
        host: reviews
        subset: v3
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

# ServiceEntry

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



