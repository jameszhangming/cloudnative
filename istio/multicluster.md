
# istio1.7 solution

部署步骤：

1. 设置istio operator的meshExpansion为开，并使用istioctl进行安装。
2. 创建虚拟机对应的namespace、service account，并根据service account生成token，拿到istiod的ca证书。在这里用官方的方法拿token没成功，后来使用了k8s官方的service account token。。
3. 配置入拦截端口和出拦截的CIDR段。
4. 配置host，将istiod的域名指向ingress地址。
5. 启动istio。

局限性。

 一台虚拟机只应该部署一个业务实例。

 需要手动指定入拦截端口和出拦截的CIDR段。

 需要手动配置istiod和业务的host。

 官方安装包支持的系统有限，目前为Debian和CentOS8(经试验，CentOS8安装有问题)。

 看似步骤简单，但是省略了很多细节，虚拟机的部署并没有想象的那么方便。