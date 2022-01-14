# istioctl proxy-config

从 Envoy 实例中获取代理配置方面的信息。

```
istioctl proxy-config <cluster|listener|route|bootstap> <pod-name>
```

## bootstap

在指定 Pod 中获取 Envoy 实例的启动信息。

基本用法：

```
istioctl proxy-config bootstrap <pod-name> [选项]

选项                    缩写    描述
--output <string>       -o      输出格式，可选 json 或者 short（缺省值 short）
```

## listener

从选定 Pod 的 Envoy 中获取监听器信息。

```
$ istioctl proxy-config listener <pod-name> [选项]

选项                    缩写    描述
--address <string>              使用 address 对监听器进行过滤（缺省值 ''）
--output <string>       -o      输出格式，可选 json 或者 short（缺省值 short）
--port <int>                    使用 port 对监听器进行过滤（缺省值 0）
--type <string>                 使用 type 对监听器进行过滤（缺省值 ''）
```

## route

获取最后发送和最后确认的从 Pilot 到网格中每个 Envoy 的 xDS 同步信息。









