
# istio1.7 solution

�����裺

1. ����istio operator��meshExpansionΪ������ʹ��istioctl���а�װ��
2. �����������Ӧ��namespace��service account��������service account����token���õ�istiod��ca֤�顣�������ùٷ��ķ�����tokenû�ɹ�������ʹ����k8s�ٷ���service account token����
3. ���������ض˿ںͳ����ص�CIDR�Ρ�
4. ����host����istiod������ָ��ingress��ַ��
5. ����istio��

�����ԡ�

 һ̨�����ֻӦ�ò���һ��ҵ��ʵ����

 ��Ҫ�ֶ�ָ�������ض˿ںͳ����ص�CIDR�Ρ�

 ��Ҫ�ֶ�����istiod��ҵ���host��

 �ٷ���װ��֧�ֵ�ϵͳ���ޣ�ĿǰΪDebian��CentOS8(�����飬CentOS8��װ������)��

 ���Ʋ���򵥣�����ʡ���˺ܶ�ϸ�ڣ�������Ĳ���û���������ô���㡣