# Multi-Primary





## 部署

Follow this guide to install the Istio control plane on both `cluster1` and `cluster2`, making each a primary cluster. Both clusters reside on the `network1` network, meaning there is direct connectivity between the pods in both clusters.

In this configuration, each control plane observes the API Servers in both clusters for endpoints.

Service workloads communicate directly (pod-to-pod) across cluster boundaries.

### Configure `cluster1` as a primary

Create the Istio configuration for `cluster1`:

```
$ cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF
```

Apply the configuration to `cluster1`:

```
$ istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml
```

### Configure `cluster2` as a primary

Create the Istio configuration for `cluster2`:

```
$ cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network1
EOF
```

Apply the configuration to `cluster2`:

```
$ istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
```

### Enable Endpoint Discovery

Install a remote secret in `cluster2` that provides access to `cluster1`’s API server.

```
$ istioctl x create-remote-secret \
    --context="${CTX_CLUSTER1}" \
    --name=cluster1 | \
    kubectl apply -f - --context="${CTX_CLUSTER2}"
```

Install a remote secret in `cluster1` that provides access to `cluster2`’s API server.

```
$ istioctl x create-remote-secret \
    --context="${CTX_CLUSTER2}" \
    --name=cluster2 | \
    kubectl apply -f - --context="${CTX_CLUSTER1}"
```

You successfully installed an Istio mesh across multiple primary clusters!

# Primary-Remote



## 部署

Follow this guide to install the Istio control plane on `cluster1` (the primary cluster) and configure `cluster2` (the remote cluster) to use the control plane in `cluster1`. Both clusters reside on the `network1` network, meaning there is direct connectivity between the pods in both clusters.

In this configuration, cluster `cluster1` will observe the API Servers in both clusters for endpoints. In this way, the control plane will be able to provide service discovery for workloads in both clusters.

Service workloads communicate directly (pod-to-pod) across cluster boundaries.

Services in `cluster2` will reach the control plane in `cluster1` via a dedicated gateway for [east-west](https://en.wikipedia.org/wiki/East-west_traffic) traffic.

### Configure `cluster1` as a primary

Create the Istio configuration for `cluster1`:

```
$ cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF
```

Apply the configuration to `cluster1`:

```
$ istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml
```

### Install the east-west gateway in `cluster1`

Install a gateway in `cluster1` that is dedicated to [east-west](https://en.wikipedia.org/wiki/East-west_traffic) traffic. By default, this gateway will be public on the Internet. Production systems may require additional access restrictions (e.g. via firewall rules) to prevent external attacks. Check with your cloud vendor to see what options are available.

```
$ samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster cluster1 --network network1 | \
    istioctl --context="${CTX_CLUSTER1}" install -y -f -
```

