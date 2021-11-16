# �ܹ��ݽ�

## istio1.1 �ܹ�

Istio ���������߼��Ϸ�Ϊ����ƽ��Ϳ���ƽ�档

* ����ƽ����һ���� sidecar ��ʽ��������ܴ���Envoy����ɡ���Щ������Ե��ںͿ���΢���� Mixer ֮�����е�����ͨ�š�

* ����ƽ�渺���������ô�����·���������������ƽ������ Mixer ��ʵʩ���Ժ��ռ�ң�����ݡ�

![istio1.1 arch](images/istio1.1 arch.jpeg "istio1.1 arch")

### Envoy

Istio ʹ�� Envoy �������չ�汾��Envoy ���� C++ �����ĸ����ܴ������ڵ���������������з����������վ�ͳ�վ������Envoy ��������ù��ܱ� Istio ���������磺

* ��̬������
* ���ؾ���
* TLS ��ֹ
* HTTP/2 & gRPC ����
* �۶���
* ������顢���ڰٷֱ�������ֵĻҶȷ���
* ����ע��
* �ḻ�Ķ���ָ��

### Mixer

Mixer ��һ��������ƽ̨������������ڷ���������ִ�з��ʿ��ƺ�ʹ�ò��ԣ����� Envoy ��������������ռ�ң�����ݡ�������ȡ�������ԣ����͵� Mixer �����������й�������ȡ�Ͳ��������ĸ�����Ϣ����μ� Mixer ���á�

Mixer �а���һ�����Ĳ��ģ�ͣ�ʹ���ܹ����뵽�������������ͻ�����ʩ��ˣ�����Щϸ���г���� Envoy ����� Istio ����ķ���

### Pilot

Pilot Ϊ Envoy sidecar �ṩ�����ֹ��ܣ�Ϊ����·�ɣ����� A/B ���ԡ���˿ȸ����ȣ��͵��ԣ���ʱ�����ԡ��۶����ȣ��ṩ���������ܡ���������������Ϊ�ĸ߼�·�ɹ���ת��Ϊ�ض��� Envoy �����ã���������ʱ�����Ǵ����� sidecar��

Pilot ��ƽ̨�ض��ķ����ֻ��Ƴ��󻯲�����ϳ�Ϊ���� Envoy ����ƽ�� API ���κ� sidecar ������ʹ�õı�׼��ʽ��������ɢ���ʹ�� Istio �ܹ��ڶ��ֻ��������У����磬Kubernetes��Consul��Nomad����ͬʱ�������������������ͬ�������档

### Citadel

Citadel ͨ��������ݺ�ƾ֤������ǿ��ķ����������û������֤����������������������δ���ܵ���������Ϊ��ά��Ա�ṩ���ڷ����ʶ������������Ƶ�ǿ��ִ�в��Ե��������� 0.5 �汾��ʼ��Istio ֧�ֻ��ڽ�ɫ�ķ��ʿ��ƣ��Կ���˭���Է������ķ��񣬶����ǻ��ڲ��ȶ���������Ĳ������ʶ��

### Galley

Galley ���������� Istio ����ƽ�������������֤�û���д�� Istio API ���á�����ʱ������ƣ�Galley ���ӹ� Istio ��ȡ���á�����ͷ�������Ķ������Ρ��������������� Istio �����ӵײ�ƽ̨������ Kubernetes����ȡ�û����õ�ϸ���и��뿪����

## istio1.5 �ܹ�

![istio1.5 arch](images/istio1.5 arch.png "istio1.5 arch")

# ����������

## galley

### galley �ܹ�

���ڵ�Galley ��������ԡ����á���������ʱ��֤, istio ����������������ȥlist/watch ���Թ�ע�ġ����á���

![galley arch](images/galley arch.png "galley arch")

Խ��Խ���Ҹ��ӵġ����á���istio �û���������಻��, ��Ҫ������:

* �����á���ȱ��ͳһ����, ������Զ���, ȱ��ͳһ�ع�����, �����������Զ�λ
* �����á��ɸ��öȵ�, ������1.1֮ǰ, ÿ��mixer adpater ����Ҫ������µ�CRD
* �����á��ĸ���, ACL ����, һ����, ����̶�, ���л��ȵ����ⶼ����̫��������

����istio���ܵ��ݽ�, ��Ԥ����istio CRD���������������, �����ƻ���Galley ǿ��Ϊistio �����á����Ʋ�, Galley ���˼����ṩ�����á���֤������, �����ṩ���ù�����ˮ��, ��������, ת��, �ַ�, �Լ��ʺ�istio������ġ����á��ַ�Э��(MCP)��

### galley validate

Galley ʹ����k8s�ṩ����һ��Admission Webhooks: ValidatingWebhook, �������õ���֤:

![galley validate](images/galley validate.png "galley validate")

istio ��Ҫһ������ValidatingWebhook��������, ���ڸ���k8s api server, ��ЩcrdӦ�÷����ĸ�������ĸ��ӿ�ȥ����֤, ��������Ϊistio-galley, �򻯵���������:

```CLI
$ kubectl get ValidatingWebhookConfiguration istio-galley -oyaml

apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: istio-galley
webhooks:- clientConfig:
  ......
    service:
      name: istio-galley
      namespace: istio-system
      path: /admitpilot
  failurePolicy: Fail
  name: pilot.validation.istio.io
  rules:
  ...pilot��ע��CRD...
    - gateways
    - virtualservices
  ......  
- clientConfig:
  ......
    service:
      name: istio-galley
      namespace: istio-system
      path: /admitmixer
  name: mixer.validation.istio.io
  rules:
  ...mixer��ע��CRD...
    - rules
    - metrics
  ......  
```

���Կ���, �����ý�pilot��mixer��ע��CRD, �ֱ𷢵��˷���istio-galley��/admitpilot��/admitmixer, ��Galley Դ���п��Ժ������ҵ���2��path Handler����ڡ�

### MCP Э��

MCP �ṩ��һ�����ö��ĺͷַ���API, ��MCP��, ���Գ�������ģ��:

* source: �����á����ṩ��, ��Istio��Galley ����source
* sink: �����á������Ѷ�, ��isito�е��͵�sink����Pilot��Mixer���
* resource: source��sink��ע����Դ��, Ҳ����isito�еġ����á�

��sink��source֮�佨���˶�ĳЩresource�Ķ��ĺͷַ���ϵ��, source �Ὣָ��resource�ı仯��Ϣ���͸�sink, sink�˿���ѡ����ܻ��߲�����resource����(�����ʽ��������), ����Ӧ����ACK/NACK ��source�ˡ�

MCP �ṩ��gRPC ��ʵ��, ���а���2��services: ResourceSource �� ResourceSink, ͨ�������, source ����Ϊ gRPC��server ��, �ṩResourceSource����, sink ��Ϊ gRPC�Ŀͻ���, sink����������������source; �����еĳ�����, source ����ΪgRPC��client��, sink��ΪgRPC��server���ṩResourceSink����, source����������������sink��

����2������, �ڲ������߼�����һ�µ�, ����sink��Ҫ����source�����resource, ����������Ķ������������������

���嵽istio�ĳ�����:

* �ڵ�k8s��Ⱥ��istio mesh��, GalleyĬ��ʵ����ResourceSource service, Pilot��Mixer����Ϊ��service��client��������Galley�������ö��ġ�
* Galley ��������ȥ��������Զ�̵�����sink, ����˵�ڶ�k8s��Ⱥ��mesh��, ����Ⱥ�е�Galley����Ϊ�����Ⱥ��Pilot/Mixer�ṩ���ù���, �缯Ⱥ��Pilot/Mixer�޷�������������ȺGalley, ��ʱ��Galley�Ϳ�����ΪgRPC��client ������������, �缯Ⱥ��Pilot/Mixer��ΪgRPC server ʵ��ResourceSink����



