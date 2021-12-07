# 虚拟机安装



## 先决条件

1. [下载 Istio 发行版](https://istio.io/latest/zh/docs/setup/getting-started/#download)
2. 执行必要的[平台安装](https://istio.io/latest/zh/docs/setup/platform-setup/)
3. 检查 [Pod 和服务的需求](https://istio.io/latest/zh/docs/ops/deployment/requirements/)
4. 虚拟机必须 IP 连通到目标网格的入口网关，如果有更高的性能需求，也可通过三层网络连通网格中的每个 Pod。
5. 阅读[虚拟机架构](https://istio.io/latest/zh/docs/ops/deployment/vm-architecture/)来理解 Istio 虚拟机集成的高级架构。

# 单一网络安装

## 准备指导环境

1. 创建虚拟机
2. 设置环境变量 `VM_APP`、`WORK_DIR`、`VM_NAMESPACE`、和 `SERVICE_ACCOUNT`（例如： `WORK_DIR="${HOME}/vmintegration"`）:

```
$ VM_APP="<将在这台虚机上运行的应用名>"
$ VM_NAMESPACE="<您的服务所在的命名空间>"
$ WORK_DIR="<证书工作目录>"
$ SERVICE_ACCOUNT="<为这台虚机提供的 Kubernetes 的服务账号名称>"
$ CLUSTER_NETWORK=""
$ VM_NETWORK=""
$ CLUSTER="Kubernetes"
```

3. 创建工作目录：

```
$ mkdir -p "${WORK_DIR}"
```

## 安装 Istio 控制平面

安装 Istio，打开控制平面的对外访问，以便您的虚拟机可以访问它。

1. 为安装创建 `IstioOperator` 空间

   ```
   $ cat <<EOF > ./vm-cluster.yaml
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   metadata:
     name: istio
   spec:
     values:
       global:
         meshID: mesh1
         multiCluster:
           clusterName: "${CLUSTER}"
         network: "${CLUSTER_NETWORK}"
   EOF
   ```

2. 安装 Istio。

```
$ istioctl install
```

3. 部署东西向网关：

```
$ samples/multicluster/gen-eastwest-gateway.sh --single-cluster | istioctl install -y -f -
```

4. 使用东西向网关暴露控制平面:

```
$ kubectl apply -f samples/multicluster/expose-istiod.yaml

# expose-istiod.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istiod-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        name: tls-istiod
        number: 15012
        protocol: tls
      tls:
        mode: PASSTHROUGH        
      hosts:
        - "*"
    - port:
        name: tls-istiodwebhook
        number: 15017
        protocol: tls
      tls:
        mode: PASSTHROUGH          
      hosts:
        - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: istiod-vs
spec:
  hosts:
  - "*"
  gateways:
  - istiod-gateway
  tls:
  - match:
    - port: 15012
      sniHosts:
      - "*"
    route:
    - destination:
        host: istiod.istio-system.svc.cluster.local
        port:
          number: 15012
  - match:
    - port: 15017
      sniHosts:
      - "*"
    route:
    - destination:
        host: istiod.istio-system.svc.cluster.local
        port:
          number: 443
```

## 配置虚拟机的命名空间

1. 创建用于托管虚拟机的名称空间：

   ```
   $ kubectl create namespace "${VM_NAMESPACE}"
   ```

2. 为虚拟机创建 ServiceAccount：

   ```
   $ kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
   ```

## 创建要传输到虚拟机的文件

首先，为虚拟机创建 `WorkloadGroup` 模板：

```
$ cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${VM_NETWORK}"
EOF
```

接下来，使用 `istioctl x workload entry` 命令来生成:

- `cluster.env`: 包含用来识别名称空间、服务帐户、网络 CIDR、和入站端口(可选)的元数据。
- `istio-token`: 用来从 CA 获取证书的 Kubernetes 令牌。
- `mesh.yaml`: 提供 `ProxyConfig` 来配置 `discoveryAddress`, 健康检查, 以及一些认证操作。
- `root-cert.pem`: 用于认证的根证书。
- `hosts`: `/etc/hosts` 的补充，代理将使用该补充从 Istiod 获取 xDS.*。

$ istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"

## 配置虚拟机

在要添加到 Istio 网格的虚拟机上，运行以下命令：

1. 将文件从 `"${WORK_DIR}"` 安全上传到虚拟机。如何安全的传输这些文件，这需要考虑到您的信息安全策略。本指南为方便起见，将所有必备文件上传到虚拟机中的 `"${HOME}"` 目录。

2. 将根证书安装到目录 `/etc/certs`:

   ```
   $ sudo mkdir -p /etc/certs
   $ sudo cp "${HOME}"/root-cert.pem /etc/certs/root-cert.pem
   ```

   

3. 将令牌安装到目录 `/var/run/secrets/tokens`:

   ```
   $ sudo  mkdir -p /var/run/secrets/tokens
   $ sudo cp "${HOME}"/istio-token /var/run/secrets/tokens/istio-token
   ```

   

4. 安装包含 Istio 虚拟机集成运行时（runtime）的包：

1. ```
   $ curl -LO https://storage.googleapis.com/istio-release/releases/1.12.0/deb/istio-sidecar.deb
   $ sudo dpkg -i istio-sidecar.deb
   ```

   5. 将 `cluster.env` 安装到目录 `/var/lib/istio/envoy/` 中:

   ```
   $ sudo cp "${HOME}"/cluster.env /var/lib/istio/envoy/cluster.env
   ```

   

2. 将网格配置文件 [Mesh Config](https://istio.io/latest/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig) 安装到目录 `/etc/istio/config/mesh`:

   ```
   $ sudo cp "${HOME}"/mesh.yaml /etc/istio/config/mesh
   ```

   

3. 将 istiod 主机添加到 `/etc/hosts`:

   ```
   $ sudo sh -c 'cat $(eval echo ~$SUDO_USER)/hosts >> /etc/hosts'
   ```

   

4. 把文件 `/etc/certs/` 和 `/var/lib/istio/envoy/` 的所有权转移给 Istio proxy:

   ```
   $ sudo mkdir -p /etc/istio/proxy
   $ sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
   ```

   ## 在虚拟机中启动 Istio

   1. 启动 Istio 代理:

      ```
      $ sudo systemctl start istio
      ```