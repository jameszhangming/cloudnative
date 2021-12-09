# Ingress Gateway

除了支持 Kubernetes [Ingress](https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/kubernetes-ingress/)，Istio还提供了另一种配置模式，[Istio Gateway](https://istio.io/latest/zh/docs/reference/config/networking/gateway/)。与 `Ingress` 相比，`Gateway` 提供了更广泛的自定义和灵活性，并允许将 Istio 功能（例如监控和路由规则）应用于进入集群的流量。

## 确定 Ingress IP 和端口

执行如下指令，确定您的 Kubernetes 集群是否运行在支持外部负载均衡的环境中：

```
$ kubectl get svc istio-ingressgateway -n istio-system

NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)   AGE
istio-ingressgateway   LoadBalancer   172.21.109.129   130.211.10.121   ...       17h
```

如果 `EXTERNAL-IP` 值已设置，说明环境正在使用外部负载均衡，可以用其为 Ingress Gateway 提供服务。如果 `EXTERNAL-IP` 值为 `<none>` （或持续显示 `<pending>`），说明环境没有为 Ingress Gateway 提供外部负载均衡，无法使用 Ingress Gateway。在这种情况下，您可以使用服务的 [Node Port](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) 访问网关。

### external load balancernode port

若已确定您的环境使用了外部负载均衡器，执行如下指令。

设置 Ingress IP 和端口：

```
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
$ export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].port}')
```

在特定的环境下，可能会使用主机名来暴露负载均衡器，而不是 IP 地址。此时，Ingress Gateway 的 `EXTERNAL-IP` 值将不再是 IP 地址，而是主机名。前文设置 `INGRESS_HOST` 环境变量的命令将执行失败。使用下面的命令更正 `INGRESS_HOST` 值：

```
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

### node port

若您的环境未使用外部负载均衡器，需要通过 Node Port 访问。执行如下命令。

设置 Ingress 端口：

```
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
$ export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].nodePort}')
```

基于集群供应商，设置 Ingress IP：

1. *GKE：*

```
$ export INGRESS_HOST=worker-node-address
```

需要创建防火墙规则，允许 TCP 流量通过 *ingressgateway* 服务的端口。执行下面的命令，设置允许流量通过 HTTP 端口、HTTPS 安全端口，或均可：

```
$ gcloud compute firewall-rules create allow-gateway-http --allow "tcp:$INGRESS_PORT"
$ gcloud compute firewall-rules create allow-gateway-https --allow "tcp:$SECURE_INGRESS_PORT"
```

2. *Minikube：*

```
$ export INGRESS_HOST=$(minikube ip)
```

## 使用 Istio Gateway 配置 Ingress

Ingress [Gateway](https://istio.io/latest/zh/docs/reference/config/networking/gateway/) 描述运行在网格边界的负载均衡器，负责接收传入的 HTTP/TCP 连接。其中配置了对外暴露的端口、协议等。但是，与[Kubernetes Ingress 资源](https://kubernetes.io/docs/concepts/services-networking/ingress/)不同，Ingress Gateway 不包含任何流量路由配置。Ingress 流量的路由使用 Istio 路由规则来配置，和内部服务请求完全一样。

1. 创建 Istio `Gateway`：

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
EOF
```

2. 为通过 `Gateway` 的入口流量配置路由：

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway        # vs生效的网关
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

已为 `httpbin` 服务创建了[Virtual Service](https://istio.io/latest/zh/docs/reference/config/networking/virtual-service/)配置，包含两个路由规则，允许流量流向路径 `/status` 和 `/delay`。

[Gateways](https://istio.io/latest/zh/docs/reference/config/networking/virtual-service/#VirtualService-gateways) 列表规约了哪些请求允许通 `httpbin-gateway` 网关。 所有其他外部请求均被拒绝并返回 404 响应。

3. 使用 *curl* 访问 *httpbin* 服务：

```
$ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/status/200"

HTTP/1.1 200 OK
server: istio-envoy
...
```

注意上文命令使用 `-H` 标识将 HTTP 头部参数 *Host* 设置为 “httpbin.example.com”。该操作为必须操作，因为 Ingress `Gateway` 已被配置用来处理 “httpbin.example.com” 的服务请求，而在测试环境中并没有为该主机绑定 DNS，而是简单直接地向 Ingress IP 发送请求。

4. 访问其他没有被显式暴露的 URL 时，将看到 HTTP 404 错误：

```
$ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/headers"

HTTP/1.1 404 Not Found
...
```

## 通过浏览器访问 Ingress 服务

在浏览器中输入 `httpbin` 服务的 URL 不能获得有效的响应，因为无法像 `curl` 那样，将请求头部参数 *Host* 传给浏览器。在现实场景中，这并不是问题，因为您需要合理配置被请求的主机及可解析的 DNS，从而在 URL 中使用主机的域名，譬如：`https://httpbin.example.com/status/200`。

为了在简单的测试和演示中绕过这个问题，请在 `Gateway` 和 `VirtualService` 配置中使用通配符 `*`。譬如，修改 Ingress 配置如下：

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

此时，便可以在浏览器中输入包含 `$INGRESS_HOST:$INGRESS_PORT` 的 URL。譬如，输入`http://$INGRESS_HOST:$INGRESS_PORT/headers`，将显示浏览器发送的所有 Header 信息。

# Secure Gateways

Secure Gateways 使用 TLS 或 mTLS 公开安全的 HTTPS 服务。



