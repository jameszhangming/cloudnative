# �������ػ���

��ͼչʾ���� productpage ����������� http://reviews.default.svc.cluster.local:9080/������������ reviews �����ڲ�ʱ��reviews �����ڲ��� Envoy Sidecar ��������������غ�·��ת���ġ�

Init ��������ʱ�����в�����ָ���� REDIRECT ģʽ�����ֻ������ NAT ����������ǲ鿴 nat ���еĹ����������������а��� ISTIO ǰ׺������ Init ����ע��ģ�����ƥ���Ǹ���������ʾ��˳����ִ�еģ����л��ж����ת��

![bookinfo traffic interception](images/bookinfo traffic interception.png "bookinfo traffic interception")

```CLI
$ iptables -t nat -L -v
# PREROUTING ��������Ŀ���ַת����DNAT������������վ TCP ������ת�� ISTIO_INBOUND ����
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target         prot opt in     out     source               destination
    2   120 ISTIO_INBOUND  tcp  --  any    any     anywhere             anywhere

# INPUT ���������������ݰ����� TCP ���������� OUTPUT ��
Chain INPUT (policy ACCEPT 2 packets, 120 bytes)
 pkts bytes target     prot opt in     out     source               destination

# OUTPUT ���������г�վ���ݰ���ת�� ISTIO_OUTPUT ����
Chain OUTPUT (policy ACCEPT 41146 packets, 3845K bytes)
 pkts bytes target        prot opt in     out     source               destination
   93  5580 ISTIO_OUTPUT  tcp  --  any    any     anywhere             anywhere

# POSTROUTING �����������ݰ���������ʱ��Ҫ�Ƚ���POSTROUTING �����ں˸������ݰ�Ŀ�ĵ��ж��Ƿ���Ҫת����ȥ�����ǿ����˴�δ���κδ���
Chain POSTROUTING (policy ACCEPT 41199 packets, 3848K bytes)
 pkts bytes target     prot opt in     out     source               destination

# ISTIO_INBOUND ����������Ŀ�ĵ�Ϊ 9080 �˿ڵ���վ�����ض��� ISTIO_IN_REDIRECT ����
Chain ISTIO_INBOUND (1 references)
 pkts bytes target             prot opt in     out     source               destination
    2   120 ISTIO_IN_REDIRECT  tcp  --  any    any     anywhere             anywhere             tcp dpt:9080

# ISTIO_IN_REDIRECT ���������е���վ������ת�����ص� 15001 �˿ڣ����˳ɹ��������������� Envoy 
Chain ISTIO_IN_REDIRECT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    2   120 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001

# ISTIO_OUTPUT ����ѡ����Ҫ�ض��� Envoy�������أ� �ĳ�վ���������з� localhost ������ȫ��ת���� ISTIO_REDIRECT��Ϊ�˱��������ڸ� Pod ������ѭ�������е� istio-proxy �û��ռ�����������ص����ĵ��õ��е���һ�����򣬱����м� OUTPUT ������Ϊ���� ISTIO_OUTPUT ����֮��ͽ�����һ���� POSTROUTING�����Ŀ�ĵط� localhost ����ת�� ISTIO_REDIRECT��������������� istio-proxy �û��ռ�ģ���ô�������������������ĵ���������ִ����һ������OUPT ����һ������������������д��������еķ� istio-proxy �û��ռ��Ŀ�ĵ��� localhost ����������ת�� ISTIO_REDIRECT��
# ���Ŀ�ĵط� localhost ����ת�� ISTIO_REDIRECT ��
# �������� istio-proxy �û��ռ�ķ� localhost ������ת�����ĵ��õ� OUTPUT ����ִ�� OUTPUT ������һ��������Ϊ OUTPUT ����û����һ�������ˣ����Ի����ִ�� POSTROUTING ��Ȼ������ iptables��ֱ�ӷ���Ŀ�ĵ�
# ��������������� istio-proxy �û��ռ䣬���Ƕ� localhost �ķ��ʣ���ô������ iptables��ֱ�ӷ���Ŀ�ĵ�
# ���������������ת�� ISTIO_REDIRECT ��
Chain ISTIO_OUTPUT (1 references)
 pkts bytes target          prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  any    lo      anywhere            !localhost
   40  2400 RETURN          all  --  any    any     anywhere             anywhere             owner UID match istio-proxy
    0     0 RETURN          all  --  any    any     anywhere             anywhere             owner GID match istio-proxy    
    0     0 RETURN          all  --  any    any     anywhere             localhost
   53  3180 ISTIO_REDIRECT  all  --  any    any     anywhere             anywhere

# ISTIO_REDIRECT ���������������ض��� Envoy�������أ� �� 15001 �˿�
Chain ISTIO_REDIRECT (2 references)
 pkts bytes target     prot opt in     out     source               destination
   53  3180 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001
```

# ��� Inbound Handler

Inbound handler �������ǽ� iptables ���ص��� downstream ������ת���� localhost���� Pod �ڵ�Ӧ�ó��������������ӡ�

**1. �鿴 reviews �ļ������Ļ���ժҪ**

```CLI
$ istioctl proxy-config listeners reviews-v1-76474f6fb7-pmglr

ADDRESS            PORT      TYPE 
172.33.3.3         9080      HTTP <--- �������� Inbound HTTP �������õ�ַ��Ϊ��ǰ Pod �� IP ��ַ
10.254.0.1         443       TCP  <--+
10.254.4.253       80        TCP     |
10.254.4.253       8080      TCP     |
10.254.109.182     443       TCP     |
10.254.22.50       15011     TCP     |
10.254.22.50       853       TCP     |
10.254.79.114      443       TCP     | 
10.254.143.179     15011     TCP     |
10.254.0.2         53        TCP     | ������ 0.0.0.0_15001 ��������Ե� Outbound �� HTTP ����
10.254.22.50       443       TCP     |
10.254.16.64       42422     TCP     |
10.254.127.202     16686     TCP     |
10.254.22.50       31400     TCP     |
10.254.22.50       8060      TCP     |
10.254.169.13      14267     TCP     |
10.254.169.13      14268     TCP     |
10.254.32.134      8443      TCP     |
10.254.118.196     443       TCP  <--+
0.0.0.0            15004     HTTP <--+
0.0.0.0            8080      HTTP    |
0.0.0.0            15010     HTTP    | 
0.0.0.0            8088      HTTP    |
0.0.0.0            15031     HTTP    |
0.0.0.0            9090      HTTP    | 
0.0.0.0            9411      HTTP    | ������ 0.0.0.0_15001 ��Ե� Outbound HTTP ����
0.0.0.0            80        HTTP    |
0.0.0.0            15030     HTTP    |
0.0.0.0            9080      HTTP    |
0.0.0.0            9093      HTTP    |
0.0.0.0            3000      HTTP    |
0.0.0.0            8060      HTTP    |
0.0.0.0            9091      HTTP <--+    
0.0.0.0            15001     TCP  <--- �������о� iptables ���ص� Inbound �� Outbound ������ת�����������������
```

������ productpage �������ִ� reviews Pod ��ʱ���Ѿ���downstream ������ȷ֪�� Pod �� IP ��ַΪ 172.33.3.3 ���ԲŻ���ʸ� Pod�����Ը������� 172.33.3.3:9080��

�Ӹ� Pod �� Listener �б��п��Կ�����0.0.0.0:15001/TCP �� Listener����ʵ�������� virtual���������е� Inbound �����������Ǹ� Listener ����ϸ���á�

```JSON
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
```

**2. virtual Listener**

UseOriginalDst���������п��Կ��� useOriginalDst ����ָ��Ϊ true������һ������ֵ��ȱʡΪ false��ʹ�� iptables �ض�������ʱ��proxy ���յĶ˿ڿ�����ԭʼĿ�ĵ�ַ�Ķ˿ڲ�һ������˴� proxy ���յĶ˿�Ϊ 15001����ԭʼĿ�ĵض˿�Ϊ 9080�����˱�־����Ϊ true ʱ��Listener �������ض�����ԭʼĿ�ĵ�ַ������ Listener���˴�Ϊ 172.33.3.3:9080�����û����ԭʼĿ�ĵ�ַ������ Listener���������ɽ������� Listener �������� virtual Listener������ envoy.tcp_proxy ����������ת���� BlackHoleCluster����� Cluster �����������������֣��� Envoy �Ҳ���ƥ������������ʱ���ͻὫ�����͸����������� 404��������������ᵽ�� Listener ������ bindToPort ���Ӧ��

```CLI
$ istioctl proxy-config listeners reviews-v1-76474f6fb7-pmglr --port 15001 -o json

[
    {
        "name": "virtual",    # ���������
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

UseOriginalDst���������п��Կ��� useOriginalDst ����ָ��Ϊ true������һ������ֵ��ȱʡΪ false��ʹ�� iptables �ض�������ʱ��proxy ���յĶ˿ڿ�����ԭʼĿ�ĵ�ַ�Ķ˿ڲ�һ������˴� proxy ���յĶ˿�Ϊ 15001����ԭʼĿ�ĵض˿�Ϊ 9080�����˱�־����Ϊ true ʱ��Listener �������ض�����ԭʼĿ�ĵ�ַ������ Listener���˴�Ϊ 172.30.135.40:9080�����û����ԭʼĿ�ĵ�ַ������ Listener���������ɽ������� Listener �������� virtual Listener������ envoy.tcp_proxy ����������ת���� BlackHoleCluster����� Cluster �����������������֣��� Envoy �Ҳ���ƥ������������ʱ���ͻὫ�����͸����������� 404��������������ᵽ�� Listener ������ bindToPort ���Ӧ��

�ò���������������ʹ��ԭʼĿ�ĵ�ַ�� Listener filter ������ò�������Ҫ��;�ǣ�Envoy ͨ������ 15001 �˿ڽ� iptables ���ص������������� Listener ���������ֱ��ת����ȥ��

**3. Listener 172.33.3.3_9080**

����˵������ Inbound handler �������� virtual Listener ת�Ƶ� 172.33.3.3_9080 Listener�������ڲ鿴�¸� Listener ���á�

```CLI
$ istioctl pc listener reviews-v1-cb8655c75-b97zc --address 172.33.3.3 --port 9080 -o json

[{
    "name": "172.33.3.3_9080",
    "address": {
        "socketAddress": {
            "address": "172.33.3.3",
            "portValue": 9080
        }
    },
    "filterChains": [
        {
            "filterChainMatch": {
                "transportProtocol": "raw_buffer"
            },
            "filters": [
                {
                    "name": "envoy.http_connection_manager",
                    "config": {
                        ... 
                        "route_config": {
                            "name": "inbound|9080||reviews.default.svc.cluster.local",
                            "validate_clusters": false,
                            "virtual_hosts": [
                                {
                                    "domains": [
                                        "*"
                                    ],
                                    "name": "inbound|http|9080",
                                    "routes": [
                                        {
                                            ...
                                            "route": {
                                                "cluster": "inbound|9080||reviews.default.svc.cluster.local",
                                                "max_grpc_timeout": "0.000s",
                                                "timeout": "0.000s"
                                            }
                                        }
                                    ]
                                }
                            ]
                        },
                        "use_remote_address": false,
                        ...
                    }
                }
            ]��
            "deprecatedV1": {
                "bindToPort": false
            }
        ...
        },
        {
            "filterChainMatch": {
                "transportProtocol": "tls"
            },
            "tlsContext": {...
            },
            "filters": [...
            ]
        }
    ],
...
}]
```

bindToPort��ע��������һ�� bindToPort �����ã���ֵΪ false�������õ�ȱʡֵΪ true����ʾ�� Listener �󶨵��˿��ϣ��˴�����Ϊ false ��� Listener ֻ�ܴ������� Listener ת�ƹ�������������������˵�� virtual Listener��

���ǿ����е� filterChains.filters �е� envoy.http_connection_manager ���ò��֣������ñ�ʾ������ת���� Cluster inbound|9080||reviews.default.svc.cluster.local ����

**4. Cluster inbound|9080||reviews.default.svc.cluster.local**

```CLI
$ istioctl pc cluster reviews-v1-cb8655c75-b97zc --fqdn reviews.default.svc.cluster.local --direction inbound -o json

[
    {
        "name": "inbound|9080||reviews.default.svc.cluster.local",
        "connectTimeout": "1.000s",
        "hosts": [
            {
                "socketAddress": {
                    "address": "127.0.0.1",
                    "portValue": 9080
                }
            }
        ],
        "circuitBreakers": {
            "thresholds": [
                {}
            ]
        }
    }
]
```

���Կ����� Cluster �� Endpoint ֱ�Ӷ�Ӧ�ľ��� localhost���پ��� iptables ת�������ͱ�Ӧ�ó������������ˡ�


# ��� Outbound Handler

��Ϊ reviews ���� ratings ������ HTTP ��������ĵ�ַ�ǣ�http://ratings.default.svc.cluster.local:9080/��Outbound handler �������ǽ� iptables ���ص��ı���Ӧ�ó��򷢳������������� Envoy �ж����·�ɵ� upstream��

Ӧ�ó�����������������Ϊ Outbound �������� iptables �ٳֺ�ת�Ƹ� Envoy Outbound handler ����Ȼ�󾭹� virtual Listener��0.0.0.0_9080 Listener��Ȼ��ͨ�� Route 9080 �ҵ� upstream �� cluster������ͨ�� EDS �ҵ� Endpoint ִ��·�ɶ�����

**1. 9080 Listener**

���ǵ������ǵ� 9080 �˿ڵ� HTTP ��վ��������ζ�������л��� 0.0.0.0:9080 �����������Ȼ�󣬴˼������������õ� RDS �в���·�����á�

```CLI
$ istioctl proxy-config listeners reviews-v1-76474f6fb7-pmglr --address 0.0.0.0 --port 9080 -o json

...
"rds": {
    "config_source": {
        "ads": {}
    },
    "route_config_name": "9080"	   # route����
}
...
```

**2. 9080 Route**

��Ϊ Envoy ����� HTTP header �е� domains ��ƥ�� VirtualHost����������ֻ�о��� ratings.default.svc.cluster.local:9080 ��һ�� VirtualHost��
```CLI
$ istioctl proxy-config routes reviews-v1-cb8655c75-b97zc --name 9080 -o json

[{
    "name": "ratings.default.svc.cluster.local:9080",
    "domains": [
        "ratings.default.svc.cluster.local",
        "ratings.default.svc.cluster.local:9080",
        "ratings",
        "ratings:9080",
        "ratings.default.svc.cluster",
        "ratings.default.svc.cluster:9080",
        "ratings.default.svc",
        "ratings.default.svc:9080",
        "ratings.default",
        "ratings.default:9080",
        "10.254.234.130",
        "10.254.234.130:9080"
    ],
    "routes": [
        {
            "match": {
                "prefix": "/"
            },
            "route": {
                "cluster": "outbound|9080||ratings.default.svc.cluster.local",
                "timeout": "0.000s",
                "maxGrpcTimeout": "0.000s"
            },
            "decorator": {
                "operation": "ratings.default.svc.cluster.local:9080/*"
            },
            "perFilterConfig": {...
            }
        }
    ]
},
..]
```

�Ӹ� Virtual Host �����п��Կ���������·�ɵ� Cluster outbound|9080||ratings.default.svc.cluster.local��

**3. Endpoint  outbound|9080||ratings.default.svc.cluster.local**

Istio 1.1 ��ǰ�汾��֧��ʹ�� istioctl ����ֱ�Ӳ�ѯ Cluster �� Endpoint������ʹ�ò�ѯ Pilot �� debug �˵�ķ�ʽ���С�

endpoints.json �ļ��а��������� Cluster �� Endpoint ��Ϣ������ֻѡȡ���е� outbound|9080||ratings.default.svc.cluster.local Cluster �Ľ�����¡�

```CLI
$ kubectl exec reviews-v1-cb8655c75-b97zc -c istio-proxy curl http://istio-pilot.istio-system.svc.cluster.local:9093/debug/edsz > endpoints.json

{
  "clusterName": "outbound|9080||ratings.default.svc.cluster.local",
  "endpoints": [
    {
      "locality": {

      },
      "lbEndpoints": [						# endpoints�б�
        {
          "endpoint": {
            "address": {
              "socketAddress": {
                "address": "172.33.100.2",
                "portValue": 9080
              }
            }
          },
          "metadata": {
            "filterMetadata": {
              "istio": {
                  "uid": "kubernetes://ratings-v1-8558d4458d-ns6lk.default"
                }
            }
          }
        }
      ]
    }
  ]
}
```

# ��������������ı仯

**1. ����������routes**

```CLI
[{
    "name": "ratings.default.svc.cluster.local:9080",
    "domains": [
        "ratings.default.svc.cluster.local",
        "ratings.default.svc.cluster.local:9080",
        "ratings",
        "ratings:9080",
        "ratings.default.svc.cluster",
        "ratings.default.svc.cluster:9080",
        "ratings.default.svc",
        "ratings.default.svc:9080",
        "ratings.default",
        "ratings.default:9080",
        "10.254.234.130",
        "10.254.234.130:9080"
    ],
    "routes": [
        {
            "match": {
                "prefix": "/"
            },
            "route": {
                "cluster": "outbound|9080||ratings.default.svc.cluster.local",
                "timeout": "0.000s",
                "maxGrpcTimeout": "0.000s"
            },
            "decorator": {
                "operation": "ratings.default.svc.cluster.local:9080/*"
            },
            "perFilterConfig": {...
            }
        }
    ]
},
..]
```

**2. ���VirtualService**

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
        subset: v1			# ����reviews��·�ɵ�v1�汾
EOF
```

**3. ������������routes**

```CLI
[{
    "name": "ratings.default.svc.cluster.local:9080",
    "domains": [
        "ratings.default.svc.cluster.local",
        "ratings.default.svc.cluster.local:9080",
        "ratings",
        "ratings:9080",
        "ratings.default.svc.cluster",
        "ratings.default.svc.cluster:9080",
        "ratings.default.svc",
        "ratings.default.svc:9080",
        "ratings.default",
        "ratings.default:9080",
        "10.254.234.130",
        "10.254.234.130:9080"
    ],
    "routes": [
        {
            "match": {
                "prefix": "/"
            },
            "route": {
                "cluster": "outbound|9080|v1|ratings.default.svc.cluster.local",      # ���ӵİ汾��
                "timeout": "0.000s",
                "maxGrpcTimeout": "0.000s"
            },
            "decorator": {
                "operation": "ratings.default.svc.cluster.local:9080/*"
            },
            "perFilterConfig": {...
            }
        }
    ]
},
..]
```

�Ա�û���� VirtualService ֮ǰ��·�ɣ�����·�ɵ� cluster �ֶε�ֵ�Ѿ���֮ǰ�� outbound|9080|ratings.default.svc.cluster.local ��Ϊ outbound|9080|v1|ratings.default.svc.cluster.local��

����Գ�������һ����û�� outbound|9080|v1|ratings.default.svc.cluster.local �����Ⱥ������������⣬�㽫�Ҳ��� SUBSET=v1 �ļ�Ⱥ��

**4. ���DestinationRule**

```CLI
$ cat <<EOF | istioctl create -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
EOF
```

 ��ʵ DestinationRule ӳ�䵽 Envoy �������ļ��о��� Cluster��������Ӧ���ܿ��� SUBSET=v1 �� Cluster �ˡ�
 
 **5. �鿴Cluster**

```CLI
$ istioctl proxy-config clusters reviews-v1-76474f6fb7-pmglr --fqdn ratings.default.svc.cluster.local --subset=v1 -o json

[
    {
        "name": "outbound|9080|v1|ratings.default.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|9080|v1|ratings.default.svc.cluster.local"
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

������һ����һ�н����ˣ����������͸�֮ǰ����·һ���ˣ������ Endpoint ��Ӧ���˱�ǩ version=v1 �� Pod��



