# 


# iptables ����

![iptables](images/iptables.png "iptables")

```CLI

```

# bookinfoӦ�� ��ʼ envoy ����

1. �� productpage Ϊ�����Ȳ鿴 productpage �� Pod Name��

```CLI
$ kubectl get pod -l app=productpage

NAME                              READY     STATUS    RESTARTS   AGE
productpage-v1-76474f6fb7-pmglr   2/2       Running   0          7h
```

2. �鿴 productpage �ļ������Ļ���ժҪ��

```CLI
$ istioctl proxy-config listeners productpage-v1-76474f6fb7-pmglr

ADDRESS            PORT      TYPE
172.30.135.40      9080      HTTP    // �� Receives all inbound traffic on 9080 from listener `0.0.0.0_15001`
10.254.223.255     15011     TCP <---+
10.254.85.22       20001     TCP     |
10.254.149.167     443       TCP     |
10.254.14.157      42422     TCP     |
10.254.238.17      9090      TCP     |  �� Receives outbound non-HTTP traffic for relevant IP:PORT pair from listener `0.0.0.0_15001`
10.254.184.32      5556      TCP     |
10.254.0.1         443       TCP     |
10.254.52.199      8080      TCP     |
10.254.118.224     443       TCP <---+  
0.0.0.0            15031     HTTP <--+
0.0.0.0            15004     HTTP    |
0.0.0.0            9093      HTTP    |
0.0.0.0            15030     HTTP    |
0.0.0.0            8080      HTTP    |  �� Receives outbound HTTP traffic for relevant port from listener `0.0.0.0_15001`
0.0.0.0            8086      HTTP    |
0.0.0.0            9080      HTTP    |
0.0.0.0            15010     HTTP <--+
0.0.0.0            15001     TCP     // �� Receives all inbound and outbound traffic to the pod from IP tables and hands over to virtual listener
```

Istio ���������µļ�������

* �� 0.0.0.0:15001 �ϵļ��������ս��� Pod ������������Ȼ�������ƽ��������������
* �� ÿ�� Service IP ����һ�������������ÿ����վ TCP/HTTPS ����һ���� HTTP ��������
* �� ÿ�� Pod ��վ������¶�Ķ˿�����һ�������������
* �� ÿ����վ HTTP ������ HTTP 0.0.0.0 �˿�����һ�������������

3. ÿ�� Sidecar ����һ���󶨵� 0.0.0.0:15001 �ļ�������IP tables �� pod ��������վ�ͳ�վ����·�ɵ�����˼������� useOriginalDst ����Ϊ true������ζ���������󽻸����������ԭʼĿ��ļ�����������Ҳ����κ�ƥ�����������������Ὣ�����͸����� 404 �� BlackHoleCluster��

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

4. ���ǵ������ǵ� 9080 �˿ڵ� HTTP ��վ��������ζ�������л��� 0.0.0.0:9080 �����������Ȼ�󣬴˼������������õ� RDS �в���·�����á�

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

5. 9080 ·�����ý�Ϊÿ�������ṩ�������������ǵ���������ǰ�� reviews ������� Envoy ��ѡ�����ǵ���������ƥ�������������һ��������ƥ�䣬Envoy �����������ƥ��ĵ�һ��·��������������£�����û���κθ߼�·�ɣ����ֻ��һ��·��ƥ���������ݡ�����·�ɸ��� Envoy �������͵� outbound|9080||reviews.default.svc.cluster.local ��Ⱥ��

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

6. �˼�Ⱥ����Ϊ�� Pilot��ͨ�� ADS�����������Ķ˵㡣��ˣ�Envoy ��ʹ�� serviceName �ֶ������Ҷ˵��б��������������һ���˵㡣

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

# bookinfoӦ�� ��� ·���������

1. ���ȴ���һ�� VirtualService��

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

2. �鿴productpage ·�����á�

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

����·�ɵ� cluster �ֶε�ֵ�Ѿ���֮ǰ�� outbound|9080|reviews.default.svc.cluster.local ��Ϊ outbound|9080|v1|reviews.default.svc.cluster.local��

3. Ϊ��ʹ���洴����·�ɿɴ������Ҫ����һ�� DestinationRule��

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

4. ��ʵ DestinationRule ӳ�䵽 Envoy �������ļ��о��� Cluster��������Ӧ���ܿ��� SUBSET=v1 �� Cluster �ˣ�

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

5. ������һ����һ�н����ˣ����������͸�֮ǰ����·һ���ˣ������ Endpoint ��Ӧ���˱�ǩ version=v1 �� Pod��

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


