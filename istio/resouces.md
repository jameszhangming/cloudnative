# Autoscaling 


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
       serverCertificate: /tmp/tls.crt
       privateKey: /tmp/tls.key
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



