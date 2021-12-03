# 访问外部服务

由于默认情况下，来自 Istio-enable Pod 的所有出站流量都会重定向到其 Sidecar 代理，集群外部 URL 的可访问性取决于代理的配置。默认情况下，Istio 将 Envoy 代理配置为允许传递未知服务的请求。尽管这为入门 Istio 带来了方便，但是，通常情况下，配置更严格的控制是更可取的。

istio支持三种访问外部服务的方法：

1. 允许 Envoy 代理将请求传递到未在网格内配置过的服务。
2. 配置 [service entries](https://istio.io/latest/zh/docs/reference/config/networking/service-entry/) 以提供对外部服务的受控访问。
3. 对于特定范围的 IP，完全绕过 Envoy 代理。

## Envoy 转发流量到外部服务

Istio 有一个[安装选项](https://istio.io/latest/zh/docs/reference/config/installation-options/)， `global.outboundTrafficPolicy.mode`，它配置 sidecar 对外部服务（那些没有在 Istio 的内部服务注册中定义的服务）的处理方式。如果这个选项设置为 `ALLOW_ANY`，Istio 代理允许调用未知的服务。如果这个选项设置为 `REGISTRY_ONLY`，那么 Istio 代理会阻止任何没有在网格中定义的 HTTP 服务或 service entry 的主机。`ALLOW_ANY` 是默认值，不控制对外部服务的访问，方便你快速地评估 Istio。

1. 要查看这种方法的实际效果，你需要确保 Istio 的安装配置了 `meshConfig.outboundTrafficPolicy.mode` 选项为 `ALLOW_ANY`。它在默认情况下是开启的，除非你在安装 Istio 时显式地将它设置为 `REGISTRY_ONLY`。

运行以下命令以确认`meshConfig.outboundTrafficPolicy.mode`设置为`ALLOW_ANY`或被省略：

```
$ kubectl get istiooperator installed-state -n istio-system -o jsonpath='{.spec.meshConfig.outboundTrafficPolicy.mode}'

ALLOW_ANY
```

您应该看到`ALLOW_ANY`或没有任何输出（默认为`ALLOW_ANY`）

2. 从 `SOURCE_POD` 向外部 HTTPS 服务发出两个请求，确保能够得到状态码为 `200` 的响应：

```
$ kubectl exec "$SOURCE_POD" -c sleep -- curl -sSI https://www.google.com | grep  "HTTP/"; kubectl exec "$SOURCE_POD" -c sleep -- curl -sI https://edition.cnn.com | grep "HTTP/"

HTTP/2 200
HTTP/2 200
```

你已经成功地从网格中发送了 egress 流量。

## 控制对外部服务的访问

使用 Istio `ServiceEntry` 配置，你可以从 Istio 集群中访问任何公开的服务。本节将向你展示如何在不丢失 Istio 的流量监控和控制特性的情况下，配置对外部 HTTP 服务([httpbin.org](http://httpbin.org/)) 和外部 HTTPS 服务([www.google.com](https://www.google.com/)) 的访问。

### 更改为默认的封锁策略

为了演示如何控制对外部服务的访问，你需要将 `global.outboundTrafficPolicy.mode` 选项，从 `ALLOW_ANY`模式 改为 `REGISTRY_ONLY` 模式。

备注：你可以向已经在 `ALLOW_ANY` 模式下的可访问服务添加访问控制。通过这种方式，你可以在一些外部服务上使用 Istio 的特性，而不会阻止其他服务。一旦你配置了所有服务，就可以将模式切换到 `REGISTRY_ONLY` 来阻止任何其他无意的访问。

1. 执行以下命令来将 `global.outboundTrafficPolicy.mode` 选项改为 `REGISTRY_ONLY`：

   如果您使用 `IstioOperator` CR 安装 Istio，请在配置中添加以下字段：

   ```yaml
   spec:
     meshConfig:
       outboundTrafficPolicy:
         mode: REGISTRY_ONLY
   ```

   否则，将等效设置添加到原始 `istioctl install` 命令中，例如：

   ```
   $ istioctl install <flags-you-used-to-install-Istio> \
                      --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
   ```

2. 从 `SOURCE_POD` 向外部 HTTPS 服务发出几个请求，验证它们现在是否被阻止：

   ```
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.google.com | grep  "HTTP/"; kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://edition.cnn.com | grep "HTTP/"
   
   command terminated with exit code 35
   command terminated with exit code 35
   ```

### 访问一个外部的 HTTP 服务

1. 创建一个 `ServiceEntry`，以允许访问一个外部的 HTTP 服务：

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS            
  location: MESH_EXTERNAL    # 网格外部服务
EOF
```

`DNS` 解析在下面的服务条目中用作安全措。将解析设置为 `NONE`会开启了攻击的可能。恶意客户端在真正连接到其他IP时，可能会伪装设置 `HOST` 头信息为 `httpbin.org`（与 `httpbin.org` 不相关）。Istio sidecar 代理将信任 HOST 头信息，并错误地允许通信，甚至将其传递到其他主机的 IP 地址。 该主机可能是恶意的站点，或者网格安全策略禁止的站点。

使用 `DNS` 解析，Sidecar 代理将忽略原始目标 IP 地址并引导流量到 `httpbin.org`，并执行 DNS 查询以获取 `httpbin.org` 的IP地址。

2. 从 `SOURCE_POD` 向外部的 HTTP 服务发出一个请求：

```
$  kubectl exec -it $SOURCE_POD -c sleep -- curl http://httpbin.org/headers

{
  "headers": {
  "Accept": "*/*",
  "Connection": "close",
  "Host": "httpbin.org",
  ...
  "X-Envoy-Decorator-Operation": "httpbin.org:80/*",  # Istio sidecar 代理添加的 header
  }
}
```

3. 检查 `SOURCE_POD` 的 sidecar 代理的日志:

```
$  kubectl logs $SOURCE_POD -c istio-proxy | tail

[2019-01-24T12:17:11.640Z] "GET /headers HTTP/1.1" 200 - 0 599 214 214 "-" "curl/7.60.0" "17fde8f7-fa62-9b39-8999-302324e6def2" "httpbin.org" "35.173.6.94:80" outbound|80||httpbin.org - 35.173.6.94:80 172.30.109.82:55314 -
```

注意与 HTTP 请求相关的 `httpbin.org/headers`.

### 访问外部 HTTPS 服务

1. 创建一个 `ServiceEntry`，允许对外部服务的访问。

   ```
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: google
   spec:
     hosts:
     - www.google.com
     ports:
     - number: 443
       name: https
       protocol: HTTPS
     resolution: DNS
     location: MESH_EXTERNAL
   EOF
   ```

2. 从 `SOURCE_POD` 往外部 HTTPS 服务发送请求：

   ```
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.google.com | grep  "HTTP/"
   
   HTTP/2 200
   ```

1. 检查 `SOURCE_POD` 的 sidecar 代理的日志：

   ```
   $ kubectl logs $SOURCE_POD -c istio-proxy | tail
   
   [2019-01-24T12:48:54.977Z] "- - -" 0 - 601 17766 1289 - "-" "-" "-" "-" "172.217.161.36:443" outbound|443||www.google.com 172.30.109.82:59480 172.217.161.36:443 172.30.109.82:59478 www.google.com
   ```

   请注意与您对 `www.google.com` 的 HTTPS 请求相关的条目。

### 管理到外部服务的流量

使用 `kubectl` 设置调用外部服务 `httpbin.org` 的超时时间为 3 秒。

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  http:
  - timeout: 3s		# 增加超时控制
    route:
      - destination:
          host: httpbin.org
        weight: 100
EOF
```

## 直接访问外部服务

如果要让特定范围的 IP 完全绕过 Istio，则可以配置 Envoy sidecars 以防止它们[拦截](https://istio.io/latest/zh/docs/concepts/traffic-management/)外部请求。要设置绕过 Istio，请更改 `global.proxy.includeIPRanges` 或 `global.proxy.excludeIPRanges` [configuration option](https://archive.istio.io/v1.4/docs/reference/config/installation-options/)，并使用 `kubectl apply` 命令更新 `istio-sidecar-injector` 配置。也可以通过设置相应的[注解](https://istio.io/latest/zh/docs/reference/config/annotations/)）在pod上进行配置，例如`traffic.sidecar.istio.io / includeOutboundIPRanges`。`istio-sidecar-injector` 配置的更新，影响的是新部署应用的 pod。

排除所有外部 IP 重定向到 Sidecar 代理的一种简单方法是将 `global.proxy.includeIPRanges` 配置选项设置为内部集群服务使用的 IP 范围。这些 IP 范围值取决于集群所在的平台。

### 配置代理绕行

使用平台的 IP 范围更新 `istio-sidecar-injector` 的配置。比如，如果 IP 范围是 10.0.0.1/24，则使用一下命令：

```
$ istioctl install <flags-you-used-to-install-Istio> --set values.global.proxy.includeIPRanges="10.0.0.1/24"
```

在 [安装 Istio](https://istio.io/latest/zh/docs/setup/install/istioctl) 命令的基础上增加 `--set values.global.proxy.includeIPRanges="10.0.0.1/24"`

### 访问外部服务

由于绕行配置仅影响新的部署，因此您需要重新部署 `sleep` 程序。

在更新 `istio-sidecar-injector` configmap 和重新部署 `sleep` 程序后，Istio sidecar 将仅拦截和管理集群中的内部请求。 任何外部请求都会绕过 Sidecar，并直接到达其预期的目的地。举个例子：

```
$ kubectl exec "$SOURCE_POD" -c sleep -- curl -sS http://httpbin.org/headers

{
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    ...
  }
}
```

与通过 HTTP 和 HTTPS 访问外部服务不同，你不会看到任何与 Istio sidecar 有关的请求头， 并且发送到外部服务的请求既不会出现在 Sidecar 的日志中，也不会出现在 Mixer 日志中。 绕过 Istio sidecar 意味着你不能再监视对外部服务的访问。

### 清除对外部服务的直接访问

更新配置，以针对各种 IP 停止绕过 sidecar 代理：

```
$ istioctl install <flags-you-used-to-install-Istio>
```