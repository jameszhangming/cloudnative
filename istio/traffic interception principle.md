# 


# iptables 规则

![iptables](images/iptables.png "iptables")

```CLI

```

# bookinfo应用 初始 envoy 配置

1. 以 productpage 为例，先查看 productpage 的 Pod Name。

```CLI
$ kubectl get pod -l app=productpage

NAME                              READY     STATUS    RESTARTS   AGE
productpage-v1-76474f6fb7-pmglr   2/2       Running   0          7h
```

2. 查看 productpage 的监听器的基本摘要。

```CLI
$ istioctl proxy-config listeners productpage-v1-76474f6fb7-pmglr

ADDRESS            PORT      TYPE
172.30.135.40      9080      HTTP    // ③ Receives all inbound traffic on 9080 from listener `0.0.0.0_15001`
10.254.223.255     15011     TCP <---+
10.254.85.22       20001     TCP     |
10.254.149.167     443       TCP     |
10.254.14.157      42422     TCP     |
10.254.238.17      9090      TCP     |  ② Receives outbound non-HTTP traffic for relevant IP:PORT pair from listener `0.0.0.0_15001`
10.254.184.32      5556      TCP     |
10.254.0.1         443       TCP     |
10.254.52.199      8080      TCP     |
10.254.118.224     443       TCP <---+  
0.0.0.0            15031     HTTP <--+
0.0.0.0            15004     HTTP    |
0.0.0.0            9093      HTTP    |
0.0.0.0            15030     HTTP    |
0.0.0.0            8080      HTTP    |  ④ Receives outbound HTTP traffic for relevant port from listener `0.0.0.0_15001`
0.0.0.0            8086      HTTP    |
0.0.0.0            9080      HTTP    |
0.0.0.0            15010     HTTP <--+
0.0.0.0            15001     TCP     // ① Receives all inbound and outbound traffic to the pod from IP tables and hands over to virtual listener
```

Istio 会生成以下的监听器：

* ① 0.0.0.0:15001 上的监听器接收进出 Pod 的所有流量，然后将请求移交给虚拟监听器。
* ② 每个 Service IP 配置一个虚拟监听器，每个出站 TCP/HTTPS 流量一个非 HTTP 监听器。
* ③ 每个 Pod 入站流量暴露的端口配置一个虚拟监听器。
* ④ 每个出站 HTTP 流量的 HTTP 0.0.0.0 端口配置一个虚拟监听器。

3. 每个 Sidecar 都有一个绑定到 0.0.0.0:15001 的监听器，IP tables 将 pod 的所有入站和出站流量路由到这里。此监听器把 useOriginalDst 设置为 true，这意味着它将请求交给最符合请求原始目标的监听器。如果找不到任何匹配的虚拟监听器，它会将请求发送给返回 404 的 BlackHoleCluster。

```CLI
$ istioctl proxy-config listeners productpage-v1-76474f6fb7-pmglr --port 15001 -o json

[
    {
        "name": "virtual",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 15001
            }
        },
        "filterChains": [
            {
                "filters": [
                    {
                        "name": "envoy.tcp_proxy",
                        "config": {
                            "cluster": "BlackHoleCluster",
                            "stat_prefix": "BlackHoleCluster"
                        }
                    }
                ]
            }
        ],
        "useOriginalDst": true
    }
]
```

4. 我们的请求是到 9080 端口的 HTTP 出站请求，这意味着它被切换到 0.0.0.0:9080 虚拟监听器。然后，此监听器在其配置的 RDS 中查找路由配置。

```CLI
$ istioctl proxy-config listeners productpage-v1-76474f6fb7-pmglr --address 0.0.0.0 --port 9080 -o json

...
"rds": {
    "config_source": {
        "ads": {}
    },
    "route_config_name": "9080"
}
...
```

5. 9080 路由配置仅为每个服务提供虚拟主机。我们的请求正在前往 reviews 服务，因此 Envoy 将选择我们的请求与域匹配的虚拟主机。一旦在域上匹配，Envoy 会查找与请求匹配的第一条路径。在这种情况下，我们没有任何高级路由，因此只有一条路由匹配所有内容。这条路由告诉 Envoy 将请求发送到 outbound|9080||reviews.default.svc.cluster.local 集群。

```CLI
$ istioctl proxy-config routes productpage-v1-76474f6fb7-pmglr --name 9080 -o json

[
    {
        "name": "9080",
        "virtualHosts": [
            {
                "name": "reviews.default.svc.cluster.local:9080",
                "domains": [
                    "reviews.default.svc.cluster.local",
                    "reviews.default.svc.cluster.local:9080",
                    "reviews",
                    "reviews:9080",
                    "reviews.default.svc.cluster",
                    "reviews.default.svc.cluster:9080",
                    "reviews.default.svc",
                    "reviews.default.svc:9080",
                    "reviews.default",
                    "reviews.default:9080",
                    "172.21.152.34",
                    "172.21.152.34:9080"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|9080||reviews.default.svc.cluster.local",
                            "timeout": "0.000s"
                        },
...
```

6. 此集群配置为从 Pilot（通过 ADS）检索关联的端点。因此，Envoy 将使用 serviceName 字段来查找端点列表并将请求代理到其中一个端点。

```CLI
$ istioctl proxy-config clusters productpage-v1-76474f6fb7-pmglr --fqdn reviews.default.svc.cluster.local -o json

[
    {
        "name": "outbound|9080||reviews.default.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|9080||reviews.default.svc.cluster.local"
        },
        "connectTimeout": "1.000s",
        "circuitBreakers": {
            "thresholds": [
                {}
            ]
        }
    }
]
```

# bookinfo应用 添加 路由治理规则

1. 首先创建一个 VirtualService。

```CLI
$ cat <<EOF | istioctl create -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
EOF
```

2. 查看productpage 路由配置。

```CLI
$ istioctl proxy-config routes productpage-v1-76474f6fb7-pmglr --name 9080 -o json

[
    {
        "name": "9080",
        "virtualHosts": [
            {
                "name": "reviews.default.svc.cluster.local:9080",
                "domains": [
                    "reviews.default.svc.cluster.local",
                    "reviews.default.svc.cluster.local:9080",
                    "reviews",
                    "reviews:9080",
                    "reviews.default.svc.cluster",
                    "reviews.default.svc.cluster:9080",
                    "reviews.default.svc",
                    "reviews.default.svc:9080",
                    "reviews.default",
                    "reviews.default:9080",
                    "172.21.152.34",
                    "172.21.152.34:9080"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|9080|v1|reviews.default.svc.cluster.local",
                            "timeout": "0.000s"
                        },
...
```

现在路由的 cluster 字段的值已经从之前的 outbound|9080|reviews.default.svc.cluster.local 变为 outbound|9080|v1|reviews.default.svc.cluster.local。

3. 为了使上面创建的路由可达，我们需要创建一个 DestinationRule：

```CLI
$ cat <<EOF | istioctl create -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
EOF
```

4. 其实 DestinationRule 映射到 Envoy 的配置文件中就是 Cluster。现在你应该能看到 SUBSET=v1 的 Cluster 了：

```CLI
$ istioctl proxy-config clusters productpage-v1-76474f6fb7-pmglr --fqdn reviews.default.svc.cluster.local --subset=v1 -o json

[
    {
        "name": "outbound|9080|v1|reviews.default.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|9080|v1|reviews.default.svc.cluster.local"
        },
        "connectTimeout": "1.000s",
        "circuitBreakers": {
            "thresholds": [
                {}
            ]
        }
    }
]
```

5. 到了这一步，一切皆明了，后面的事情就跟之前的套路一样了，具体的 Endpoint 对应打了标签 version=v1 的 Pod：

```CLI
$ kubectl get pod -l app=reviews,version=v1 -o wide
NAME                          READY     STATUS    RESTARTS   AGE       IP              NODE
reviews-v1-5b487cc689-njx5t   2/2       Running   0          11h       172.30.104.38   192.168.123.248

$ curl http://$PILOT_SVC_IP:8080/debug/edsz|grep "outbound|9080|v1|reviews.default.svc.cluster.local" -A 27 -B 2
{
  "clusterName": "outbound|9080|v1|reviews.default.svc.cluster.local",
  "endpoints": [
    {
      "lbEndpoints": [
        {
          "endpoint": {
            "address": {
              "socketAddress": {
                "address": "172.30.104.38",
                "portValue": 9080
              }
            }
          },
          "metadata": {
            "filterMetadata": {
              "istio": {
                  "uid": "kubernetes://reviews-v1-5b487cc689-njx5t.default"
                }
            }
          }
        }
      ]
    }
  ]
},
```


