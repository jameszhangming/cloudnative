# Internal communication

## K8S ClusterIP

Kubernetes��Pod��ΪӦ�ò������С��λ��kubernetes�����Pod������������е��ȣ��������������١�Ǩ�ơ�ˮƽ�����ȣ����Pod ��IP��ַ���ǹ̶��ģ�������ֱ�Ӳ���Pod IP�Է�����з��ʡ�

Ϊ��������⣬Kubernetes�ṩ��Service��Դ��Service���ṩͬһ������Ķ��Pod���оۺϡ�һ��Service�ṩһ�������Cluster IP����˶�Ӧһ�����߶���ṩ�����Pod���ڼ�Ⱥ�з��ʸ�Serviceʱ������Cluster IP���ɣ�Kube-proxy���𽫷��͵�Cluster IP������ת������˵�Pod�ϡ�

## Istio Sidecar Proxy

Cluster IP����˷���֮���໥���ʵ����⣬��������Kube-proxy������ģʽ���Կ�����Cluster IP�ķ�ʽֻ�ṩ�˷����ֺͻ�����LB���ܡ����ҪΪ������ͨ��Ӧ������·�ɹ����Լ��ṩMetrics collection��distributed tracing�ȷ���ܿع���,�ͱ��������Istio�ṩ�ķ������������ˡ�

��Kubernetes�в���Istio��Istioͨ��iptables��Sidecar Proxy�ӹܷ���֮���ͨ�ţ��������໥ͨ�Ų���ͨ��Kube-proxy������ͨ��Istio��Sidecar Proxy���С����������������ģ�Client���������iptables�ض���Sidecar Proxy��Sidecar Proxy���ݴӿ������ȡ�ķ�������Ϣ��·�ɹ���ѡ��һ����˵�Server Pod�������ӣ�����ת��Client������

Istio Sidecar Proxy��Kube-proxy��userspaceģʽ�Ĺ����������ƣ�����ͨ�����û��ռ��һ��������ʵ�ֿͻ��������ת���ͺ�˶��Pod֮��ĸ��ؾ��⡣���ߵĲ�ͬ���ǣ�Kube-Proxy�������Ĳ㣬��Sidecar Proxy����һ���߲�����������HTTP��GRPS��Ӧ�ò��������д����ת������˹��ܸ�Ϊǿ�󣬿�����Ͽ�����ʵ�ָ�Ϊ����·�ɹ���ͷ���ܿع��ܡ�

![sidecar proxy](images/sidecar proxy.png "sidecar proxy")

# External communication

## K8S NodePort

Kubernetes��Pod IP��Cluster IP��ֻ���ڼ�Ⱥ�ڲ����ʣ�������ͨ����Ҫ���ⲿ�����Ϸ��ʼ�Ⱥ�е�ĳЩ����Kubernetes�ṩ��NodePort��Ϊ��Ⱥ�ṩ�ⲿ������ڡ�

NodePort�ڼ�Ⱥ�е�ÿ�������ڵ���ΪService�ṩһ������˿ڣ�����������������϶�Service���з��ʡ�

![k8s nodeport](images/k8s nodeport.png "k8s nodeport")

## K8S LoadBalance

NodePort�ṩ��һ�ִ��ⲿ�������Kubernetes��Ⱥ�ڲ�Service�ķ��������÷�����������һЩ���ƣ��������ַ�ʽ��Ҫ�����ڳ��򿪷������ʺ����ڲ�Ʒ����

* Kubernetes cluster host��IP������һ��well-known IP�����ͻ��˱���֪����IP����Cluster�е�host�Ǳ���Ϊ��Դ�ؿ����ģ���������ɾ����ÿ��host��IPһ��Ҳ�Ƕ�̬����ģ���˲�������Ϊhost IP�Կͻ��˶�����well-known IP��
* �ͻ��˷���ĳһ���̶���host IP�ķ�ʽ���ڵ�����ϡ�����һ̨host崻��ˣ�kubernetes cluster���Ӧ�� reload����һ�ڵ��ϣ����ͻ��˾��޷�ͨ����host��nodeport����Ӧ���ˡ�
* ͨ��һ�������ڵ���Ϊ������ڣ������������ϴ�ʱ��������ƿ����

Ϊ�˽����Щ���⣬Kubernetes�ṩ��LoadBalancer��ͨ����Service����ΪLoadBalancer���ͣ�Kubernetes�������ڵ��NodePortǰ�ṩ��һ���Ĳ�ĸ��ؾ����������Ĳ㸺�ؾ����������ⲿ���������ַ�������Ķ���ڵ��NodePort�˿��ϡ�

��ͼչʾ��Kubernetes���ͨ��LoadBalancer��ʽ�����ṩ������ڣ�ͼ��LoadBalancer������������������ڵ��ϵ�NodePort����˲���������Pod�ṩ���񡣸��ݼ�Ⱥ�Ĺ�ģ��������LoadBalancer������Խ������������ڵ㣬�Խ��и��ɷֵ���

![k8s loadbalance](images/k8s loadbalance.png "k8s loadbalance")

## K8S Ingress

LoadBalancer���͵�Service�ṩ�����Ĳ㸺�ؾ���������ֻ��Ҫ���Ⱪ¶һ�������ʱ�򣬲������ַ�ʽ��û������ġ�����һ��Ӧ����Ҫ�����ṩ�������ʱ�����ø÷�ʽ��Ҫ��Ϊÿһ���Ĳ����IP+Port��������һ���ⲿload balancer��

һ����˵��ͬһ��Ӧ�õĶ������/��Դ�����ͬһ�������£�����������£��������Load balancer����ȫû�б�Ҫ�ģ����������˶���Ŀ����͹���ɱ�������ֱ�ӽ�����¶���ⲿ�û�Ҳ�ᵼ����ǰ�˺ͺ�˵���ϣ�Ӱ���˺�˼ܹ�������ԣ�����Ժ�����ҵ������Է�����е�����ֱ��Ӱ�쵽�ͻ��ˡ�Ϊ�˽�������⣬����ͨ��ʹ��Kubernetes Ingress����Ϊ������ڡ�

Kubernetes Ingress������һ��Ӧ�ò㣨OSI�߲㣩�ĸ��ؾ����������Ը���HTTP��������ݽ�����ͬһ��TCP�˿ڵ�����ַ�����ͬ��Kubernetes Service���书�ܰ�����

* ��HTTP�����URL����·�ɣ�ͬһ��TCP�˿ڽ������������Ը���URL·�ɵ�Cluster�еĲ�ͬ����
* ��HTTP�����Host����·�ɣ�ͬһ��IP�������������Ը���HTTP�����Host·�ɵ�Cluster�еĲ�ͬ����

V�������ⲿ���絽��Pod������·�����£�

1. �ⲿ������ͨ���Ĳ�Load Balancer�����ڲ�����
2. Load Balancer�������ַ�����˶�������ڵ��ϵ�NodePort (userspaceת��)
3. �����NodePort���뵽Ingress Controller (iptabes����Ingress Controller������һ��NodePort���͵�Service)
4. Ingress Controller����Ingress rule�����߲�ַ�������HTTP��URL��Host������ַ�����ͬ��Service (userspaceת��)
5. Service���������յ��뵽����ṩ�����Pod�� (iptabes����)

��0.8�汾��ǰ��Istioȱʡ����K8s Ingress����ΪService Mesh��������ڡ�K8s Ingressͳһ��Ӧ�õ�������ڣ��������������⣺

K8s Ingress�Ƕ�����Istio��ϵ֮��ģ���Ҫ��������Ingress rule�������ã�����ϵͳ��ں��ڲ��������׻��������·�ɹ������ã���ά�͹����Ϊ���ӡ�

K8s Ingress rule�Ĺ��ܽ�������������ڴ�ʵ�ֺ������ڲ����Ƶ�·�ɹ���Ҳ���߱�����sidecar�������������������Դ�������ΪӦ��ϵͳʵ�ֻҶȷ������ֲ�ʽ���ٵȷ���ܿع��ܡ�

![k8s ingress for istio](images/k8s ingress for istio.png "k8s ingress for istio")

## Istio Gateway

Istio������ʶ����Ingress��Mesh�ڲ����ø��ѵ����⣬��˴�0.8�汾��ʼ������������ Gateway ��Դ����K8s Ingress����ʾ������ڡ�

Istio Gateway��Դ����ֻ������L4-L6�Ĺ��ܣ����籩¶�Ķ˿ڣ�TLS���õȣ���Gateway���ԺͰ�һ��VirtualService����VirtualService �п��������߲�·�ɹ�����Щ�߲�·�ɹ���������ݰ��շ���汾��������е���������ע�룬HTTP�ض���HTTP��д������Mesh�ڲ�֧�ֵ�·�ɹ���

Gateway��VirtualService���ڱ�ʾIstio Ingress������ģ�ͣ�Istio Ingress��ȱʡʵ��������˺�Sidecar��ͬ��Envoy proxy��

ͨ���÷�ʽ��Istio��������һ�µ�����ģ��ͬʱ������������غ��ڲ���sidecar������Щ���ð���·�ɹ��򣬲��Լ�顢Telementry�ռ��Լ���������ܿع��ܡ�

![istio gateway](images/istio gateway.png "istio gateway")

## API Gateway

����Gateway��VirtualServiceʵ�ֵ�Istio Ingress Gateway�ṩ��������ڴ��Ļ���ͨ�Ź��ܣ������ɿ���ͨ�ź�����·�ɹ��򡣵�����һ������Ӧ����˵��������ڳ��˻�����ͨѶ����֮�⣬����һЩ������Ӧ�ò㹦���������磺

* ������ϵͳ��API�ķ��ʿ���
* �û���ϵͳ�ķ��ʿ���
* �޸�����/��������
* ����API���������ڹ���
* ������ʵ�SLA���������Ʒ�

![ingress solution compare](images/ingress solution compare.png "ingress solution compare")

API Gateway�����кܴ�һ������Ҫ���ݲ�ͬ��Ӧ��ϵͳ���ж��ƣ�Ŀǰ������ʱ������ܱ�����K8s Ingress����Istio Gateway�Ĺ淶֮�С�Ϊ��������Щ����ӿ�ֳ��˸��಻ͬ��k8s Ingress Controller�Լ�Istio Ingress Gatewayʵ�֣�����Ambassador ��Kong, Traefik,Solo�ȡ�

��Щ���ز�Ʒ��ʵ�����ṩ������K8s Ingress������ͬʱ���ṩ��ǿ���API Gateway���ܣ�������ȱ��ͳһ�ı�׼����Щ��չʵ��֮���໥֮�䲢�����ݡ������ź����ǣ�Ŀǰ��ЩIngress controller����û����ʽ�ṩ��Istio �����漯�ɵ�������

## API Gateway + Sidecar Proxy

��Ŀǰ�����ҵ�һ��ͬʱ�߱�API Gateway��Isito Ingress���������ص�����£�һ�����еķ�����ʹ��API Gateway��Sidecar Proxyһ��Ϊ���������ṩ�ⲿ������ڡ�

����API Gateway�Ѿ��߱��߲����صĹ��ܣ�Mesh Ingress�е�Sidecarֻ��Ҫ�ṩVirtualService��Դ��·��������������Ҫ�ṩGateway��Դ��������������˲���Sidecar Proxy���ɡ�������ڴ���Sidecar Proxy�������ڲ�Ӧ��Pod��Sidecar Proxy��Ψһһ�������ǣ���Sidecarֻ�ӹ�API Gateway��Mesh�ڲ��������������ӹ��ⲿ����API Gateway����������Ӧ��Pod�е�Sidecar��Ҫ�ӹܽ���Ӧ�õ�����������

![API gateway and sidecar proxy](images/API gateway and sidecar proxy.png "API gateway and sidecar proxy")

����API Gateway��Sidecar Proxyһ����Ϊ���������������ڣ����ܹ�ͨ�������ؽ��ж��ƿ��������Ʒ��API���صĸ��������ֿ�����������ڴ����÷��������ṩ������·�������ͷֲ�ʽ���٣����Եȹܿع��ܣ��Ƿ��������Ʒ������ص�һ�����뷽����

���ܷ���Ŀ��ǣ�����ͼ���Կ��������ø÷������ⲿ����Ĵ�����������ڴ�������Sidecar Proxy��һ������˸÷�ʽ�����������������ʧ��������ʧ����ȫ���Խ��ܵġ�

��������ʱ�Ӷ��ԣ��ڷ��������У�һ���ⲿ��������Ҫ�����϶�Ĵ����Ӧ�ý��̵Ĵ�����Ingress������һ������������ʱ��Ӱ��������Բ��ƣ����Ҷ��ھ������Ӧ����˵������ת����ռ��ʱ����������ͺ�С��99%�ĺ�ʱ����ҵ���߼������ϵͳ�������ӵĸ�ʱ�ӷǳ����У��������¿��Ǹ�ϵͳ�Ƿ���Ҫ����΢����ܹ��ͷ�������

�������������ԣ������ڴ�����������������ƿ���������ͨ����API Gateway + Sidecar Proxy��ɵ�Ingress�������ˮƽ��չ����������������и��ɷֵ��������������ڵ�������������



