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
  - '*'     # Ĭ�������ʹ�á�*��������ζ�Ÿ�ServiceEntry������ÿ�������ռ䡣��.������������Ϊ��ǰ�����ռ䡣
  hosts:  
  - vmapp.powerful.svc.cluster.local     # ���ʵ�ַ
  location: MESH_EXTERNAL  
  ports:  
  - name: http    
    number: 80                        # ����˿�
    protocol: HTTP  
  resolution: STATIC  
  endpoints:  
    - address: 172.16.16.34       #  ������ĵ�ַ
      labels:      
        app: vmapp
        nsf.skiff.netease.com/platform: virutalmachine    
      ports:      
        http: 19990              # ������Ϸ���¶�Ķ˿�
```



