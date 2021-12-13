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

## 配置单向 TLS 入口网关

### 生成客户端和服务器证书和密钥

对于此任务，您可以使用自己喜欢的工具来生成证书和密钥。下面的命令使用[openssl](https://man.openbsd.org/openssl.1)。

1. 创建用于服务签名的根证书和私钥：

```
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
```

2. 为 `httpbin.example.com` 创建证书和私钥：

```
$ openssl req -out httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
$ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in httpbin.example.com.csr -out httpbin.example.com.crt
```

#### 配置单机TLS入口网关

1. 确定已在[准备工作](https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/ingress-control#before-you-begin)环节完成[httpbin](https://github.com/istio/istio/tree/release-1.12/samples/httpbin)服务的部署。

2. 为 Ingress Gateway 创建 Secret:

   ```
   $ kubectl create -n istio-system secret tls httpbin-credential --key=httpbin.example.com.key --cert=httpbin.example.com.crt
   ```

3. 为端口443定义一个带有 `servers:` 部分的网关，并将 `credentialName` 的值指定为 `httpbin-credential`。这些值与 Secret 名称相同。 TLS 模式的值应为 `SIMPLE`。

   ```yaml
   $ cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: mygateway
   spec:
     selector:
       istio: ingressgateway # use istio default ingress gateway
     servers:
     - port:
         number: 443
         name: https
         protocol: HTTPS
       tls:
         mode: SIMPLE
         credentialName: httpbin-credential  # must be the same as secret
       hosts:
       - httpbin.example.com
   EOF
   ```

4. 配置网关的入口流量路由，定义相应的虚拟服务。

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - mygateway
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

5. 发送 HTTPS 请求访问 `httpbin` 服务：

```
$ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
--cacert example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
```

The `httpbin` service will return the [418 I’m a Teapot](https://tools.ietf.org/html/rfc7168#section-2.3.3) code.

6. 删除网关的 secret，并创建一个新的 secret 来修改入口网关的凭据。

```
$ kubectl -n istio-system delete secret httpbin-credential
```

```
$ mkdir new_certificates
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout new_certificates/example.com.key -out new_certificates/example.com.crt
$ openssl req -out new_certificates/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout new_certificates/httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
$ openssl x509 -req -days 365 -CA new_certificates/example.com.crt -CAkey new_certificates/example.com.key -set_serial 0 -in new_certificates/httpbin.example.com.csr -out new_certificates/httpbin.example.com.crt
$ kubectl create -n istio-system secret tls httpbin-credential \
--key=new_certificates/httpbin.example.com.key \
--cert=new_certificates/httpbin.example.com.crt
```

7. `curl` 使用新证书链访问 `httpbin` 服务：

```
$ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
--cacert new_certificates/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"

...
HTTP/2 418
...
    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
```

8. 如果使用先前的证书链访问httpbin，将返回失败。

```
$ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
--cacert example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"

...
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS alert, Server hello (2):
* curl: (35) error:04FFF06A:rsa routines:CRYPTO_internal:block type is not 01
```

#### 为多个主机配置 TLS 入口网关

您可以为多个主机（例如 `httpbin.example.com` 和 `helloworld-v1.example.com` ）配置入口网关。入口网关检索与特定凭据名称相对应的唯一凭据。

1. 要恢复 httpbin 的凭据，请删除 secret 并重新创建。

```
$ kubectl -n istio-system delete secret httpbin-credential
$ kubectl create -n istio-system secret tls httpbin-credential \
--key=httpbin.example.com.key \
--cert=httpbin.example.com.crt
```

2. 启动 `helloworld-v1`

3. 为 `helloworld-v1.example.com` 生成证书和私钥：

```
$ openssl req -out helloworld-v1.example.com.csr -newkey rsa:2048 -nodes -keyout helloworld-v1.example.com.key -subj "/CN=helloworld-v1.example.com/O=helloworld organization"
$ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in helloworld-v1.example.com.csr -out helloworld-v1.example.com.crt
```

4. 创建 `helloworld-credential` secret:

```
$ kubectl create -n istio-system secret tls helloworld-credential --key=helloworld-v1.example.com.key --cert=helloworld-v1.example.com.crt
```

5. 为端口 443 定义一个包含两个 server 的网关。将每个端口上的 `credentialName` 的值分别设置为 `httpbin-credential` 和 `helloworld-credential` 。将 TLS 模式设置为 `SIMPLE`。

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mygateway
spec:
  selector:
    istio: ingressgateway # use istio default ingress gateway
  servers:
  - port:
      number: 443
      name: https-httpbin
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: httpbin-credential
    hosts:
    - httpbin.example.com
  - port:
      number: 443
      name: https-helloworld
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: helloworld-credential
    hosts:
    - helloworld-v1.example.com
EOF
```

6. 配置网关的流量路由。定义相应的虚拟服务。

```
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-v1
spec:
  hosts:
  - helloworld-v1.example.com
  gateways:
  - mygateway
  http:
  - match:
    - uri:
        exact: /hello
    route:
    - destination:
        host: helloworld-v1
        port:
          number: 5000
EOF
```

7. 发送一个 HTTPS 请求到 `helloworld-v1.example.com`:

```
$ curl -v -HHost:helloworld-v1.example.com --resolve "helloworld-v1.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
--cacert example.com.crt "https://helloworld-v1.example.com:$SECURE_INGRESS_PORT/hello"

HTTP/2 200
```

8. 发送一个 HTTPS 请求到 `httpbin.example.com`，仍然返回一个茶壶:

```
$ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
--cacert example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"

...
    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
```

## 配置双向 TLS 入口网关

您可以扩展网关的定义以支持[双向TLS](https://en.wikipedia.org/wiki/Mutual_authentication)。删除入口网关的 secret 并创建一个新的，以更改入口网关的凭据。服务器使用 CA 证书来验证其客户端，并且必须使用名称 `cacert` 来持有 CA 证书。

```
$ kubectl -n istio-system delete secret httpbin-credential
$ kubectl create -n istio-system secret generic httpbin-credential --from-file=tls.key=httpbin.example.com.key \
--from-file=tls.crt=httpbin.example.com.crt --from-file=ca.crt=example.com.crt
```

1. 更改网关的定义, 将 TLS 模式设置为 `MUTUAL` 。

```
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
 name: mygateway
spec:
 selector:
   istio: ingressgateway # use istio default ingress gateway
 servers:
 - port:
     number: 443
     name: https
     protocol: HTTPS
   tls:
     mode: MUTUAL
     credentialName: httpbin-credential # must be the same as secret
   hosts:
   - httpbin.example.com
EOF
```

2. 尝试使用先前的方法发送 HTTPS 请求，并查看失败的详情：

```
$ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
--cacert example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"

* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Request CERT (13):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Certificate (11):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* TLSv1.3 (IN), TLS alert, unknown (628):
* OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
```

3. 生成客户端证书和私钥：

```
$ openssl req -out client.example.com.csr -newkey rsa:2048 -nodes -keyout client.example.com.key -subj "/CN=client.example.com/O=client organization"
$ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in client.example.com.csr -out client.example.com.crt
```

4. 重新发送带客户端证书和私钥的 `curl` 请求。使用 –cert 标志传递客户端证书，使用 –key 标志传递私钥。

```
$ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
--cacert example.com.crt --cert client.example.com.crt --key client.example.com.key \
"https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"

...
    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
```

## 密钥格式

Istio 支持读取不同的 Secret 格式，以支持与各种工具（例如[cert-manager](https://istio.io/latest/zh/docs/ops/integrations/certmanager/))的集成：

- 如上所述，包含 `tls.key` 和 `tls.crt` 的 TLS secret。对于双向 TLS，可以使用 `ca.crt` 密钥。
- 包含 `key` 和 `cert` 的通用 Secret。对于双向 TLS，可以使用 `cacert` 密钥。
- 包含 `key` 和 `cert` 的通用 Secret。对于双向 TLS，还可以单独设置名为 `<secret>-cacert` 的通用 secret，该 secret 含 `cacert` 密钥。例如，`httpbin-credential` 包含 `key` 和 `cert`，而 `httpbin-credential-cacert` 包含 `cacert`。
- `cacert` 密钥可以是由单个 CA 证书连接组成的 CA 包。