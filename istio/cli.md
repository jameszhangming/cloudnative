# istioctl proxy-config

�� Envoy ʵ���л�ȡ�������÷������Ϣ��

```
istioctl proxy-config <cluster|listener|route|bootstap> <pod-name>
```

## bootstap

��ָ�� Pod �л�ȡ Envoy ʵ����������Ϣ��

�����÷���

```
istioctl proxy-config bootstrap <pod-name> [ѡ��]

ѡ��                    ��д    ����
--output <string>       -o      �����ʽ����ѡ json ���� short��ȱʡֵ short��
```

## listener

��ѡ�� Pod �� Envoy �л�ȡ��������Ϣ��

```
$ istioctl proxy-config listener <pod-name> [ѡ��]

ѡ��                    ��д    ����
--address <string>              ʹ�� address �Լ��������й��ˣ�ȱʡֵ ''��
--output <string>       -o      �����ʽ����ѡ json ���� short��ȱʡֵ short��
--port <int>                    ʹ�� port �Լ��������й��ˣ�ȱʡֵ 0��
--type <string>                 ʹ�� type �Լ��������й��ˣ�ȱʡֵ ''��
```

## route

��ȡ����ͺ����ȷ�ϵĴ� Pilot ��������ÿ�� Envoy �� xDS ͬ����Ϣ��









