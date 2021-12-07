# Egress Gateway

Istio 使用 [Ingress and Egress gateways](https://istio.io/latest/zh/docs/reference/config/networking/gateway/) 配置运行在服务网格边缘的负载均衡。 Ingress gateway 允许您定义网格所有入站流量的入口。Egress gateway 是一个与 Ingress gateway 对称的概念，它定义了网格的出口。Egress gateway 允许您将 Istio 的功能（例如，监视和路由规则）应用于网格的出站流量。

## 使用场景

设想一个对安全有严格要求的组织。**要求服务网格所有的出站流量必须经过一组专用节点**。专用节点运行在专门的机器上，与集群中运行应用程序的其他节点隔离。这些专用节点用于实施 egress 流量的策略，并且受到比其余节点更严密地监控。

另一个使用场景是**集群中的应用节点没有公有 IP**，所以在该节点上运行的网格 service 无法访问互联网。通过定义 egress gateway，将公有 IP 分配给 egress gateway 节点，用它引导所有的出站流量，可以使应用节点以受控的方式访问外部服务。

## 部署 Istio egress gateway

1. 检查 Istio egress gateway 是否已布署：

   ```
   $ kubectl get pod -l istio=egressgateway -n istio-system
   ```

   如果没有 pod 返回，通过接下来的步骤来部署 Istio egress gateway。

2. 如果您使用 `IstioOperator` CR安装Istio，请在配置中添加以下字段：

   ```yaml
   spec:
     components:
       egressGateways:
       - name: istio-egressgateway
         enabled: true
   ```

   否则，将等效设置添加到原始 `istioctl install` 命令中，例如：

   ```
   $ istioctl install <flags-you-used-to-install-Istio> \
                      --set components.egressGateways[0].name=istio-egressgateway \
                      --set components.egressGateways[0].enabled=true
   ```

## 定义 Egress gateway 并引导 HTTP 流量

首先创建一个 `ServiceEntry`，允许流量直接访问一个外部服务。

1. 为 `edition.cnn.com` 定义一个 `ServiceEntry`：

```
 kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: cnn
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
EOF
```

必须在服务条目中使用 `DNS` 解析。如果分辨率为 `NONE`，则网关将将流量引导到一个无限循环中。这是因为网关收到原始请求 目标 IP 地址，该地址等于网关的服务IP（因为请求是由 sidecar 定向的 网关的代理）。

借助 `DNS` 解析，网关执行 DNS 查询以获取外部服务的 IP 地址并进行定向该 IP 地址的流量。

2. 发送 HTTPS 请求到 https://edition.cnn.com/politics，验证 `ServiceEntry` 是否已正确应用。

```
$ kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - http://edition.cnn.com/politics

...
HTTP/1.1 301 Moved Permanently
...
location: https://edition.cnn.com/politics
...

HTTP/2 200
Content-Type: text/html; charset=utf-8
...
```

输出结果应该与 [发起 TLS 的 Egress 流量](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-tls-origination/) 中的 `配置对外部服务的访问` 示例相同，都还没有发起 TLS。

3. 为 `edition.cnn.com` 端口 80 创建 egress `Gateway`。并为指向 egress gateway 的流量创建一个 destination rule。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway    # 映射到egress gateway svc
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - edition.cnn.com
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-cnn
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local   # 指向Gateway的svc
  subsets:
  - name: cnn
EOF
```

要通过 egress gateway 引导多个主机，您可以在 `Gateway` 中包含主机列表，或使用 `*` 匹配所有主机。 应该将 `DestinationRule` 中的`subset` 字段用于其他主机。

4. 定义一个 `VirtualService`，将流量从 sidecar 引导至 Egress Gateway，再从 Egress Gateway 引导至外部服务：

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-cnn-through-egress-gateway
spec:
  hosts:
  - edition.cnn.com
  gateways:
  - istio-egressgateway  # egress 网关
  - mesh     # sidecar
  http:
  - match:
    - gateways:
      - mesh     # vs规则生效到sidecar
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local   # 指向 dr host，最终到网关
        subset: cnn   # 分组
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway      # vs规则生效到网关
      port: 80
    route:
    - destination:
        host: edition.cnn.com   # 指向外部服务，即 service entry host
        port:
          number: 80
      weight: 100
EOF
```

5. 再次发送 HTTP 请求到 https://edition.cnn.com/politics。

```
$ kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - http://edition.cnn.com/politics

...
HTTP/1.1 301 Moved Permanently
...
location: https://edition.cnn.com/politics
...

HTTP/2 200
Content-Type: text/html; charset=utf-8
...
```

6. 检查 `istio-egressgateway` pod 的日志，并查看与我们的请求对应的行。如果 Istio 部署在 `istio-system` 命名空间中，则打印日志的命令是：

```
$ kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail
```

你应该会看到一行类似于下面这样的内容：

```plain
[2019-09-03T20:57:49.103Z] "GET /politics HTTP/2" 301 - "-" "-" 0 0 90 89 "10.244.2.10" "curl/7.64.0" "ea379962-9b5c-4431-ab66-f01994f5a5a5" "edition.cnn.com" "151.101.65.67:80" outbound|80||edition.cnn.com - 10.244.1.5:80 10.244.2.10:50482 edition.cnn.com -
```

## 用 Egress gateway 发起 HTTPS 请求

接下来尝试使用 Egress Gateway 发起 HTTPS 请求（TLS 由应用程序发起）。您需要在相应的 `ServiceEntry`、egress `Gateway` 和 `VirtualService` 中指定 `TLS` 协议的端口 443。

1. 为 `edition.cnn.com` 定义 `ServiceEntry`：

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: cnn
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 443
    name: tls
    protocol: TLS
  resolution: DNS
EOF
```

2. 发送 HTTPS 请求到 https://edition.cnn.com/politics，验证您的 `ServiceEntry` 是否已正确生效。

```
$ kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - https://edition.cnn.com/politics

...
HTTP/2 200
Content-Type: text/html; charset=utf-8
...
```

3. 为 `edition.cnn.com` 创建一个 egress `Gateway`。除此之外还需要创建一个 destination rule 和一个 virtual service，用来引导流量通过 Egress Gateway，并通过 Egress Gateway 与外部服务通信。

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: tls
      protocol: TLS
    hosts:
    - edition.cnn.com
    tls:
      mode: PASSTHROUGH
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-cnn
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: cnn
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-cnn-through-egress-gateway
spec:
  hosts:
  - edition.cnn.com
  gateways:
  - mesh
  - istio-egressgateway
  tls:
  - match:
    - gateways:
      - mesh
      port: 443
      sni_hosts:
      - edition.cnn.com
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: cnn
        port:
          number: 443
  - match:
    - gateways:
      - istio-egressgateway
      port: 443
      sni_hosts:
      - edition.cnn.com
    route:
    - destination:
        host: edition.cnn.com
        port:
          number: 443
      weight: 100
EOF
```

4. 发送 HTTPS 请求到 https://edition.cnn.com/politics。输出结果应该和之前一样。

```
$ kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - https://edition.cnn.com/politics

...
HTTP/2 200
Content-Type: text/html; charset=utf-8
...
```

5. 检查 Egress Gateway 代理的日志。如果 Istio 部署在 `istio-system` 命名空间中，则打印日志的命令是：

```
$ kubectl logs -l istio=egressgateway -n istio-system
```

你应该会看到类似于下面的内容：

```plain
[2019-01-02T11:46:46.981Z] "- - -" 0 - 627 1879689 44 - "-" "-" "-" "-" "151.101.129.67:443" outbound|443||edition.cnn.com 172.30.109.80:41122 172.30.109.80:443 172.30.109.112:59970 edition.cnn.com
```

## 其他安全注意事项

注意，Istio 中定义的 egress `Gateway` 本身并没有为其所在的节点提供任何特殊处理。集群管理员或云提供商可以在专用节点上部署 egress gateway，并引入额外的安全措施，从而使这些节点比网格中的其他节点更安全。

另外要注意的是，Istio **无法强制** 让所有出站流量都经过 egress gateway，Istio 只是通过 sidecar 代理实现了这种流向。攻击者只要绕过 sidecar 代理，就可以不经 egress gateway 直接与网格外的服务进行通信，从而避开了 Istio 的控制和监控。出于安全考虑，集群管理员和云供应商必须确保网格所有的出站流量都要经过 egress gateway。这需要通过 Istio 之外的机制来满足这一要求。例如，集群管理员可以配置防火墙，拒绝 egress gateway 以外的所有流量。[Kubernetes 网络策略](https://kubernetes.io/docs/concepts/services-networking/network-policies/)也能禁止所有不是从 egress gateway 发起的出站流量。此外，集群管理员和云供应商还可以对网络进行限制，让运行应用的节点只能通过 gateway 来访问外部网络。要实现这一限制，可以只给 gateway Pod 分配公网 IP，并且可以配置 NAT 设备，丢弃来自 egress gateway pod 之外的所有流量。

# Egress 网关的 TLS 发起过程

本节描述如何使用 egress 网关发起与示例[为 Egress 流量发起 TLS 连接](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-tls-origination/)中一样的 TLS。 注意，这种情况下，TLS 的发起过程由 egress 网关完成，而不是像之前示例演示的那样由 sidecar 完成。

1. 为 `edition.cnn.com` 定义一个 `ServiceEntry`：

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: cnn
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 80          # 80端口
    name: http
    protocol: HTTP
  - number: 443        # 443端口
    name: https
    protocol: HTTPS
  resolution: DNS
EOF
```

2. 发送一个请求至 [http://edition.cnn.com/politics](https://edition.cnn.com/politics)，验证 `ServiceEntry` 已被正确应用。

```
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics

HTTP/1.1 301 Moved Permanently
...
location: https://edition.cnn.com/politics
...

command terminated with exit code 35
```

3. 为 *edition.cnn.com* 创建一个 egress `Gateway`，端口 443，以及一个 sidecar 请求的目标规则，sidecar 请求被直接导向 egress 网关。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway    # 关联gateway service
  servers:
  - port:
      number: 80            # 只开放了80端口，作为service开放的端口
      name: https-port-for-tls-origination
      protocol: HTTPS       # 协议类型为https
    hosts:
    - edition.cnn.com
    tls:
      mode: ISTIO_MUTUAL    
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-cnn
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local   # 导向 gateway service
  subsets:
  - name: cnn
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
      portLevelSettings:
      - port:
          number: 80              # 80端口，即gateway中定义的端口
        tls:
          mode: ISTIO_MUTUAL
          sni: edition.cnn.com
EOF
```

4. 定义一个 `VirtualService` 来引导流量流经 egress 网关，以及一个 `DestinationRule` 为访问 `edition.cnn.com` 的请求发起 TLS 连接：

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-cnn-through-egress-gateway
spec:
  hosts:
  - edition.cnn.com
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80    # 请求端口为80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local   # sidecar，导向gateway
        subset: cnn
        port:
          number: 80     # 80端口，目标服务的端口
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway    
      port: 80                  # 请求端口为80
    route:
    - destination:
        host: edition.cnn.com   # 网关，对应service entry的DR
        port:
          number: 443           # 映射DR的443端口
      weight: 100
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: originate-tls-for-edition-cnn-com
spec:
  host: edition.cnn.com
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    portLevelSettings:
    - port:
        number: 443  # 443端口配置， 映射到serviceentry的443端口
      tls:
        mode: SIMPLE # initiates HTTPS for connections to edition.cnn.com
EOF
```

5. 发送一个 HTTP 请求至 [http://edition.cnn.com/politics](https://edition.cnn.com/politics)。

```
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics

HTTP/1.1 200 OK
...
```

输出将与在示例[为 Egress 流量发起 TLS 连接](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-tls-origination/)中显示的一样，发起 TLS 连接后，不再显示 *301 Moved Permanently* 消息。

6. 检查 `istio-egressgateway` pod 的日志，将看到一行与请求相关的记录。 若 Istio 部署在 `istio-system` 命名空间中，打印日志的命令为：

```
$ kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail
```

将看到类似如下一行：

```plain
[2020-06-30T16:17:56.763Z] "GET /politics HTTP/2" 200 - "-" "-" 0 1295938 529 89 "10.244.0.171" "curl/7.64.0" "cf76518d-3209-9ab7-a1d0-e6002728ef5b" "edition.cnn.com" "151.101.129.67:443" outbound|443||edition.cnn.com 10.244.0.170:54280 10.244.0.170:8080 10.244.0.171:35628 - -
```

# 通过 egress 网关发起双向 TLS 连接

本章节描述如何配置一个 egress 网关，为外部服务发起 TLS 连接，只是这次服务要求双向 TLS。

## 生成客户端和服务器的证书与密钥

对于此任务，您可以使用自己喜欢的工具来生成证书和密钥。以下命令使用[openssl](https://man.openbsd.org/openssl.1)

1. 为您的服务签名证书创建根证书和私钥：

   ```
   $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
   ```

2. 为 `my-nginx.mesh-external.svc.cluster.local` 创建证书和私钥：

   ```
   $ openssl req -out my-nginx.mesh-external.svc.cluster.local.csr -newkey rsa:2048 -nodes -keyout my-nginx.mesh-external.svc.cluster.local.key -subj "/CN=my-nginx.mesh-external.svc.cluster.local/O=some organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in my-nginx.mesh-external.svc.cluster.local.csr -out my-nginx.mesh-external.svc.cluster.local.crt
   ```

3. 生成客户端证书和私钥：

   ```
   $ openssl req -out client.example.com.csr -newkey rsa:2048 -nodes -keyout client.example.com.key -subj "/CN=client.example.com/O=client organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in client.example.com.csr -out client.example.com.crt
   ```

## 部署一个双向 TLS 服务器

为了模拟一个真实的支持双向 TLS 协议的外部服务， 在 Kubernetes 集群中部署一个 [NGINX](https://www.nginx.com/) 服务器，该服务器运行在 Istio 服务网格之外，譬如：运行在一个没有开启 Istio sidecar proxy 注入的命名空间中。

1. 创建一个命名空间，表示 Istio 网格之外的服务，`mesh-external`。注意在这个命名空间中，sidecar 自动注入是没有[开启](https://istio.io/latest/zh/docs/setup/additional-setup/sidecar-injection/#deploying-an-app)的，不会在 pods 中自动注入 sidecar proxy。

```
$ kubectl create namespace mesh-external
```

2. 创建 Kubernetes [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) ，保存服务器和 CA 的证书。

```
$ kubectl create -n mesh-external secret tls nginx-server-certs --key my-nginx.mesh-external.svc.cluster.local.key --cert my-nginx.mesh-external.svc.cluster.local.crt    # SSL证书
$ kubectl create -n mesh-external secret generic nginx-ca-certs --from-file=example.com.crt # CA证书
```

3. 生成 NGINX 服务器的配置文件：

```
$ cat <<\EOF > ./nginx.conf
events {
}

http {
  log_format main '$remote_addr - $remote_user [$time_local]  $status '
  '"$request" $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"';
  access_log /var/log/nginx/access.log main;
  error_log  /var/log/nginx/error.log;

  server {
    listen 443 ssl;

    root /usr/share/nginx/html;
    index index.html;

    server_name my-nginx.mesh-external.svc.cluster.local;
    ssl_certificate /etc/nginx-server-certs/tls.crt;    # 服务器证书
    ssl_certificate_key /etc/nginx-server-certs/tls.key; # 服务器私钥
    ssl_client_certificate /etc/nginx-ca-certs/example.com.crt;  # client证书签发的CA
    ssl_verify_client on;   # 双向认证
  }
}
EOF
```

4. 生成 Kubernetes [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 保存 NGINX 服务器的配置文件：

```
$ kubectl create configmap nginx-configmap -n mesh-external --from-file=nginx.conf=./nginx.conf
```

5. 部署 NGINX 服务器：

```
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  namespace: mesh-external
  labels:
    run: my-nginx
spec:
  ports:
  - port: 443
    protocol: TCP
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  namespace: mesh-external
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 443   # 容器开放的端口
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx
          readOnly: true
        - name: nginx-server-certs
          mountPath: /etc/nginx-server-certs
          readOnly: true
        - name: nginx-ca-certs
          mountPath: /etc/nginx-ca-certs
          readOnly: true
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-configmap
      - name: nginx-server-certs
        secret:
          secretName: nginx-server-certs
      - name: nginx-ca-certs
        secret:
          secretName: nginx-ca-certs
EOF
```

6. 为 `nginx.example.com` 定义一个 `ServiceEntry` 和一个 `VirtualService`，指示 Istio 引导目标为 `nginx.example.com` 的流量流向 NGINX 服务器：

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: nginx
spec:
  hosts:
  - nginx.example.com
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  endpoints:
  - address: my-nginx.mesh-external.svc.cluster.local  # 第五步定义的service
    ports:
      https: 443
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx
spec:
  hosts:
  - nginx.example.com
  tls:
  - match:
    - port: 443
      sni_hosts:
      - nginx.example.com
    route:
    - destination:
        host: nginx.example.com    # 
        port:
          number: 443
      weight: 100
EOF
```

### 使用客户端证书部署 egress 网关

1. 生成 Kubernetes [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 保存客户端和 CA 的证书。

```
$ kubectl create -n istio-system secret tls nginx-client-certs --key client.example.com.key --cert client.example.com.crt
$ kubectl create -n istio-system secret generic nginx-ca-certs --from-file=example.com.crt
```

2. 部署 `istio-egressgateway` 挂载新生成的 Secrets 的 Volume。使用的参数选项与生成 `istio.yaml` 中的一致，创建下面的 `gateway-patch.json` 文件：

```
$ cat > gateway-patch.json <<EOF
[{
  "op": "add",
  "path": "/spec/template/spec/containers/0/volumeMounts/0",
  "value": {
    "mountPath": "/etc/istio/nginx-client-certs",
    "name": "nginx-client-certs",
    "readOnly": true
  }
},
{
  "op": "add",
  "path": "/spec/template/spec/volumes/0",
  "value": {
  "name": "nginx-client-certs",
    "secret": {
      "secretName": "nginx-client-certs",
      "optional": true
    }
  }
},
{
  "op": "add",
  "path": "/spec/template/spec/containers/0/volumeMounts/1",
  "value": {
    "mountPath": "/etc/istio/nginx-ca-certs",
    "name": "nginx-ca-certs",
    "readOnly": true
  }
},
{
  "op": "add",
  "path": "/spec/template/spec/volumes/1",
  "value": {
  "name": "nginx-ca-certs",
    "secret": {
      "secretName": "nginx-ca-certs",
      "optional": true
    }
  }
}]
EOF
```

3. 通过以下命令部署应用 `istio-egressgateway`：

```
$ kubectl -n istio-system patch --type=json deploy istio-egressgateway -p "$(cat gateway-patch.json)"
```

4. 验证密钥和证书被成功装载入 `istio-egressgateway` pod：

```
$ kubectl exec -n istio-system "$(kubectl -n istio-system get pods -l istio=egressgateway -o jsonpath='{.items[0].metadata.name}')" -- ls -al /etc/istio/nginx-client-certs /etc/istio/nginx-ca-certs
```

`tls.crt` 与 `tls.key` 在 `/etc/istio/nginx-client-certs` 中，而 `ca-chain.cert.pem` 在 `/etc/istio/nginx-ca-certs` 中。

### 为 egress 流量配置双向 TLS

1. 为 `my-nginx.mesh-external.svc.cluster.local` 创建一个 egress `Gateway` 端口为 443，以及目标规则和虚拟服务来引导流量流经 egress 网关并从 egress 网关流向外部服务。

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - my-nginx.mesh-external.svc.cluster.local
    tls:
      mode: ISTIO_MUTUAL
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-nginx
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: nginx
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
      portLevelSettings:
      - port:
          number: 443
        tls:
          mode: ISTIO_MUTUAL
          sni: my-nginx.mesh-external.svc.cluster.local
EOF
```

2. 定义一个 `VirtualService` 引导流量流经 egress 网关：

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-nginx-through-egress-gateway
spec:
  hosts:
  - my-nginx.mesh-external.svc.cluster.local
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: nginx
        port:
          number: 443
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 443
    route:
    - destination:
        host: my-nginx.mesh-external.svc.cluster.local
        port:
          number: 443
      weight: 100
EOF
```

3. 添加 `DestinationRule` 执行双向 TLS

```
$ kubectl apply -n istio-system -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: originate-mtls-for-nginx
spec:
  host: my-nginx.mesh-external.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    portLevelSettings:
    - port:
        number: 443
      tls:
        mode: MUTUAL
        clientCertificate: /etc/istio/nginx-client-certs/tls.crt
        privateKey: /etc/istio/nginx-client-certs/tls.key
        caCertificates: /etc/istio/nginx-ca-certs/example.com.crt
        sni: my-nginx.mesh-external.svc.cluster.local
EOF
```

4. 发送一个 HTTP 请求至 `http://my-nginx.mesh-external.svc.cluster.local`：

```
$ kubectl exec "$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -- curl -sS http://my-nginx.mesh-external.svc.cluster.local

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

5. 检查 `istio-egressgateway` pod 日志，有一行与请求相关的日志记录。 如果 Istio 部署在命名空间 `istio-system` 中，打印日志的命令为：

```
$ kubectl logs -l istio=egressgateway -n istio-system | grep 'my-nginx.mesh-external.svc.cluster.local' | grep HTTP
```

将显示类似如下的一行：

```plain
[2018-08-19T18:20:40.096Z] "GET / HTTP/1.1" 200 - 0 612 7 5 "172.30.146.114" "curl/7.35.0" "b942b587-fac2-9756-8ec6-303561356204" "my-nginx.mesh-external.svc.cluster.local" "172.21.72.197:443"
```

# Wildcard 主机的 egress

[控制 Egress 流量](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/)任务和[配置一个 Egress 网关](https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-gateway/)示例描述如何配置特定主机的 egress 流量，如：`edition.cnn.com`。本示例描述如何为通用域中的一组特定主机开启 egress 流量，譬如：`*.wikipedia.org`，无需单独配置每一台主机。

## 背景

假定您想要为 Istio 中所有语种的 `wikipedia.org` 站点开启 egress 流量。每个语种的 `wikipedia.org` 站点均有自己的主机名，譬如：英语和德语对应的主机分别为 `en.wikipedia.org` 和 `de.rikipedia.org`。您希望通过通用配置项开启所有 Wikipedia 站点的 egress 流量，无需单独配置每个语种的站点。

## 配置访问 wildcard 主机的 egress 网关

能否配置通过 egress 网关访问 wildcard 主机取决于这组 wildcard 域名有唯一一个通用主机。 以 **.wikipedia.org* 为例。每个语种特殊的站点都有自己的 *wikipedia.org* 服务器。您可以向任意一个 **.wikipedia.org* 站点的 IP 发送请求，包括 _www.wikipedia.org_，该站点[管理服务](https://en.wikipedia.org/wiki/Virtual_hosting)所有特定主机。

通常情况下，通用域中的所有域名并不由一个唯一的 hosting server 提供服务。此时，需要一个更加复杂的配置。

### 单一 hosting 服务器的 Wildcard 配置

当一台唯一的服务器为所有 wildcard 主机提供服务时，基于 egress 网关访问 wildcard 主机的配置与普通主机类似，除了：配置的路由目标不能与配置的主机相同，如：wildcard 主机，需要配置为通用域集合的唯一服务器主机。

1. 为 **.wikipedia.org* 创建一个 egress `Gateway`、一个目标规则以及一个虚拟服务，来引导请求通过 egress 网关并从 egress 网关访问外部服务。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*.wikipedia.org"
    tls:
      mode: PASSTHROUGH
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-wikipedia
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
    - name: wikipedia
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-wikipedia-through-egress-gateway
spec:
  hosts:
  - "*.wikipedia.org"
  gateways:
  - mesh
  - istio-egressgateway
  tls:
  - match:
    - gateways:
      - mesh
      port: 443
      sniHosts:
      - "*.wikipedia.org"
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: wikipedia
        port:
          number: 443
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 443
      sniHosts:
      - "*.wikipedia.org"
    route:
    - destination:
        host: www.wikipedia.org    # 目标服务
        port:
          number: 443
      weight: 100
EOF
```

2. 为目标服务器 *www.wikipedia.org* 创建一个 `ServiceEntry`。

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: www-wikipedia
spec:
  hosts:
  - www.wikipedia.org
  ports:
  - number: 443
    name: tls
    protocol: TLS
  resolution: DNS
EOF
```

3. 发送请求至 [https://en.wikipedia.org](https://en.wikipedia.org/) 和 [https://de.wikipedia.org](https://de.wikipedia.org/)：

```
$ kubectl exec -it $SOURCE_POD -c sleep -- sh -c 'curl -s https://en.wikipedia.org/wiki/Main_Page | grep -o "<title>.*</title>"; curl -s https://de.wikipedia.org/wiki/Wikipedia:Hauptseite | grep -o "<title>.*</title>"'

<title>Wikipedia, the free encyclopedia</title>
<title>Wikipedia – Die freie Enzyklopädie</title>
```

4. 检查 egress 网关代理访问 **.wikipedia.org* 的计数器统计值。如果 Istio 部署在 `istio-system` 命名空间中，打印输出计数器的命令为：

```
$ kubectl exec -it $(kubectl get pod -l istio=egressgateway -n istio-system -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -n istio-system -- pilot-agent request GET clusters | grep '^outbound|443||www.wikipedia.org.*cx_total:'

outbound|443||www.wikipedia.org::208.80.154.224:443::cx_total::2
```

### 任意域的 Wildcard 配置

前面章节的配置生效，是因为 **.wikipedia.org* 站点可以由任意一个 *wikipedia.org* 服务器提供服务。然而，情况并不总是如此。 譬如，你可能想要配置 egress 控制到更为通用的 wildcard 域，如：`*.com` 或 `*.org`。

配置流量流向任意 wildcard 域，为 Istio 网关引入一个挑战。在前面的章节中，您在配置中，将 *www.wikipedia.org* 配置为网关的路由目标主机，将流量导向该地址。 然而，网关并不知道它接收到的请求中任意 arbitrary 主机的 IP 地址。 这是 Istio 默认的 egress 网关代理 [Envoy](https://www.envoyproxy.io/) 的限制。Envoy 或者将流量路由至提前定义好的主机，或者路由至提前定义好的 IP 地址，或者是请求的最初 IP 地址。在网关案例中，请求的最初目标 IP 被丢弃，因为请求首先路由至 egress 网关，其目标 IP 为网关的 IP 地址。

因此，基于 Envoy 的 Istio 网关无法将流量路由至没有预先配置的 arbitrary 主机，从而，无法对任意 wildcard 域实施流量控制。 为了对 HTTPS 和任意 TLS 连接开启流量控制，您需要在 Envoy 的基础上再部署一个 SNI 转发代理。Envoy 将访问 wildcard 域的请求路由至 SNI 转发代理，代理反过来将请求转发给 SNI 值中约定的目标地址。