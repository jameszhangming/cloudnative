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

# Egress TLS Origination

假设有一个遗留应用正在使用 HTTP 和外部服务进行通信。而运行该应用的组织却收到了一个新的需求，该需求要求必须对所有外部的流量进行加密。 此时，使用 Istio 便可通过修改配置实现此需求，而无需更改应用中的任何代码。 该应用可以发送未加密的 HTTP 请求，然后 Istio 将为应用加密请求。

从应用源头发送未加密的 HTTP 请求并让 Istio 执行 TSL 升级的另一个好处是可以产生更好的遥测并为未加密的请求提供更多的路由控制。

## 配置对外部服务的访问

1. 创建一个 `ServiceEntry` 启用对 `edition.cnn.com` 的访问：

   ```
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: edition-cnn-com
   spec:
     hosts:
     - edition.cnn.com
     ports:
     - number: 80
       name: http-port
       protocol: HTTP
     - number: 443
       name: https-port
       protocol: HTTPS
     resolution: DNS
   EOF
   ```

2. 向外部的 HTTP 服务发送请求：

   ```
   $ kubectl exec "${SOURCE_POD}" -c sleep -- curl -sSL -o /dev/null -D - http://edition.cnn.com/politics
   
   HTTP/1.1 301 Moved Permanently
   ...
   location: https://edition.cnn.com/politics
   ...
   
   HTTP/2 200
   ...
   ```

请注意 *curl* 的 `-L` 标志，该标志指示 *curl* 将遵循重定向。 在这种情况下，服务器将对到 `http://edition.cnn.com/politics` 的 HTTP 请求返回重定向响应 ([301 Moved Permanently](https://tools.ietf.org/html/rfc2616#section-10.3.2))。 而重定向响应将指示客户端使用 HTTPS 向 `https://edition.cnn.com/politics` 重新发送请求。 对于第二个请求，服务器则返回了请求的内容和 *200 OK* 状态码。

尽管 *curl* 命令简明地处理了重定向，但是这里有两个问题。 第一个问题是请求冗余，它使获取 `http://edition.cnn.com/politics` 内容的延迟加倍。 第二个问题是 URL 中的路径（在本例中为 *politics* ）被以明文的形式发送。 如果有人嗅探您的应用与 `edition.cnn.com` 之间的通信，他将会知晓该应用获取了此网站中哪些特定的内容。 而出于隐私的原因，您可能希望阻止这些内容被披露。

通过配置 `Istio` 执行 `TLS` 发起，则可以解决这两个问题。

## 用于 egress 流量的 TLS 发起

1. 重新定义上一节的 `ServiceEntry` 和 `VirtualService` 以重写 HTTP 请求端口，并添加一个 `DestinationRule` 以执行 TLS 发起。

   ```
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: edition-cnn-com
   spec:
     hosts:
     - edition.cnn.com
     ports:
     - number: 80
       name: http-port
       protocol: HTTP
       targetPort: 443
     - number: 443
       name: https-port
       protocol: HTTPS
     resolution: DNS
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: edition-cnn-com
   spec:
     host: edition.cnn.com
     trafficPolicy:
       portLevelSettings:
       - port:
           number: 80
         tls:
           mode: SIMPLE # initiates HTTPS when accessing edition.cnn.com
   EOF
   ```

   上面的 `DestinationRule` 将对端口80和 `ServiceEntry` 上的HTTP请求执行TLS发起。然后将端口 80 上的请求重定向到目标端口 443。

1. 如上一节一样，向 `http://edition.cnn.com/politics` 发送 HTTP 请求：

   ```
   $ kubectl exec "${SOURCE_POD}" -c sleep -- curl -sSL -o /dev/null -D - http://edition.cnn.com/politics
   
   HTTP/1.1 200 OK
   ...
   ```

   这次将会收到唯一的 *200 OK* 响应。 因为 Istio 为 *curl* 执行了 TSL 发起，原始的 HTTP 被升级为 HTTPS 并转发到 `edition.cnn.com`。 服务器直接返回内容而无需重定向。 这消除了客户端与服务器之间的请求冗余，使网格保持加密状态，隐藏了您的应用获取 `edition.cnn.com` 中 *politics* 的事实。

   请注意，您使用了一些与上一节相同的命令。 您可以通过配置 Istio，使以编程方式访问外部服务的应用无需更改任何代码即可获得 TLS 发起的好处。

2. 请注意，使用 HTTPS 访问外部服务的应用程序将继续像以前一样工作：

   ```
   $ kubectl exec "${SOURCE_POD}" -c sleep -- curl -sSL -o /dev/null -D - https://edition.cnn.com/politics
   
   HTTP/2 200
   ...
   ```