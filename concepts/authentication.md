# Basic Auth

��HTTP�У�������֤��Basic access authentication����һ������������ҳ������������ͻ��˳���������ʱ�ṩ�û����Ϳ�����ʽ�����ƾ֤��һ�ֵ�¼��֤��ʽ��

**�ŵ�**

* ������֤��һ���ŵ��ǻ������������е���ҳ�������֧�ֻ�����֤��������֤�����ڿɹ������ʵĻ�������վ��ʹ�ã���ʱ�����С��˽��ϵͳ��ʹ�ã���·������ҳ����ӿڣ��������Ļ���HTTPժҪ��֤��Ϊ���������֤�������ģ�������Կ����԰�ȫ�ķ�ʽ�ڲ���ȫ��ͨ���ϴ��䡣
* ����Ա��ϵͳ����Ա��ʱ���ڿ������绷����ʹ�û�����֤��ʹ��Telnet��������������Э�鹤���ֶ��ز���Web������������һ���鷳�Ĺ��̣����������ϴ�����������˿ɶ��ģ��Ա������ϡ�

**ȱ��**

* ��Ȼ������֤�ǳ�����ʵ�֣����÷������������µļ���Ļ����ϣ������ͻ��˺ͷ���������֮��������ǰ�ȫ���ŵġ��ر��ǣ����û��ʹ��SSL/TLS�����Ĵ���㰲ȫ��Э�飬��ô�����Ĵ������Կ�Ϳ�������ױ����ء��÷���Ҳͬ��û�жԷ��������ص���Ϣ�ṩ������
* �ִ�������������֤��Ϣֱ����ǩҳ����������رգ������û������ʷ��¼��HTTPû��Ϊ�������ṩһ�ַ���ָʾ�ͻ��˶�����Щ���������Կ������ζ�ŷ����������û����ر������������£���û��һ����Ч�ķ��������û�ע����


## ������ʽ

**1��ʹ�������**

��ʹ����������������� HTTP Basic Auth �ķ�����ʱ���ᵯ���Ի��������û��������뼴�ɡ�

**2��ʹ�� HTTP Client ����**

```CLI
http://user:passwd@httpbin.org/basic-auth/user/passwd
```


## ԭ��

��һ�����͵�HTTP�ͻ��˺�HTTP�������ĶԻ�����������װ��ͬһ̨������ϣ�localhost�����������²��裺

1. �ͻ�������һ����Ҫ�����֤��ҳ�棬����û���ṩ�û����Ϳ����ͨ�����û��ڵ�ַ������һ��URL�����Ǵ���һ��ָ���ҳ������ӡ�
2. �������Ӧһ��401Ӧ���룬���ṩһ����֤��
3. �ӵ�Ӧ��󣬿ͻ�����ʾ����֤��ͨ���������ʵļ������ϵͳ�����������û�����ʾ�����û����Ϳ����ʱ�û�����ѡ��ȷ����ȡ����
4. �û��������û����Ϳ���󣬿ͻ����������ԭ�ȵ�������������֤��Ϣͷ��Ȼ�����·����ٴγ��ԡ�
   * ��������ֵ����ʽ�������ģ��ü��ܹ����ǿ��Է�������ģ�
     ```CLI
     Authorization: Basic base64encode(username+":"+password)   
     ```

ע��:�ͻ����п��ܲ���Ҫ�û��������ڵ�һ�������оͷ�����֤��Ϣͷ��

û����֤��Ϣ��������Ӧ���£�

```CLI
GET /private/index.html HTTP/1.0
Host: localhost

HTTP/1.0 401 Authorization Required
Server: HTTPd/1.0
Date: Sat, 27 Nov 2004 10:18:15 GMT
WWW-Authenticate: Basic realm="Secure Area"
Content-Type: text/html
Content-Length: 311

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
 "http://www.w3.org/TR/1999/REC-html401-19991224/loose.dtd">
<HTML>
  <HEAD>
    <TITLE>Error</TITLE>
    <META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=ISO-8859-1">
  </HEAD>
  <BODY><H1>401 Unauthorized.</H1></BODY>
</HTML>

# �����û������������������·�������
GET /private/index.html HTTP/1.0
Host: localhost
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
```

Response �е� WWW-Authenticate �ֶλ�ָʾ���������ѯ���û������������ʾ��


# Bearer Auth




# JWT


# OAuth 2.0

OAuth��һ��������Ȩ��authorization���Ŀ��������׼����ȫ����õ��㷺Ӧ�ã�Ŀǰ�İ汾��2.0�档


## Ӧ�ó���

��һ��"�Ƴ�ӡ"����վ�����Խ��û�������Google����Ƭ����ӡ�������û�Ϊ��ʹ�ø÷��񣬱�����"�Ƴ�ӡ"��ȡ�Լ�������Google�ϵ���Ƭ��

������ֻ�еõ��û�����Ȩ��Google�Ż�ͬ��"�Ƴ�ӡ"��ȡ��Щ��Ƭ����ô��"�Ƴ�ӡ"��������û�����Ȩ�أ�

��ͳ�����ǣ��û����Լ���Google�û��������룬����"�Ƴ�ӡ"�����߾Ϳ��Զ�ȡ�û�����Ƭ�ˡ����������������¼������ص�ȱ�㣺

* "�Ƴ�ӡ"Ϊ�˺����ķ��񣬻ᱣ���û������룬�����ܲ���ȫ��
* Google���ò����������¼��������֪���������������¼������ȫ��
* "�Ƴ�ӡ"ӵ���˻�ȡ�û�������Google�������ϵ�Ȩ�����û�û������"�Ƴ�ӡ"�����Ȩ�ķ�Χ����Ч�ڡ�
* �û�ֻ���޸����룬�����ջظ���"�Ƴ�ӡ"��Ȩ������������������ʹ���������л���û���Ȩ�ĵ�����Ӧ�ó���ȫ��ʧЧ��
* ֻҪ��һ��������Ӧ�ó����ƽ⣬�ͻᵼ���û�����й©���Լ����б����뱣��������й©��

OAuth����Ϊ�˽��������Щ����������ġ�


## ���ʶ���

����ϸ����OAuth 2.0֮ǰ����Ҫ�˽⼸��ר�����ʡ�

* Third-party application��������Ӧ�ó��򣬱������ֳ�"�ͻ���"��client��������һ�������е�"�Ƴ�ӡ"��
* HTTP service��HTTP�����ṩ�̣������м��"�����ṩ��"������һ�������е�Google��
* Resource Owner����Դ�����ߣ��������ֳ�"�û�"��user����
* User Agent���û����������о���ָ�������
* Authorization server����֤���������������ṩ��ר������������֤�ķ�������
* Resource server����Դ���������������ṩ�̴���û����ɵ���Դ�ķ�������������֤��������������ͬһ̨��������Ҳ�����ǲ�ͬ�ķ�������

OAuth�����þ�����"�ͻ���"��ȫ�ɿصػ�ȡ"�û�"����Ȩ����"�������ṩ��"���л�����


## OAuth��˼·

OAuth��"�ͻ���"��"�����ṩ��"֮�䣬������һ����Ȩ�㣨authorization layer����"�ͻ���"����ֱ�ӵ�¼"�����ṩ��"��ֻ�ܵ�¼��Ȩ�㣬�Դ˽��û���ͻ������ֿ�����"�ͻ���"��¼��Ȩ�����õ����ƣ�token�������û������벻ͬ���û������ڵ�¼��ʱ��ָ����Ȩ�����Ƶ�Ȩ�޷�Χ����Ч�ڡ�

"�ͻ���"��¼��Ȩ���Ժ�"�����ṩ��"�������Ƶ�Ȩ�޷�Χ����Ч�ڣ���"�ͻ���"�����û���������ϡ�


## OAuth����Ȩ����

![oauth2 flow](images/oauth2_flow.png "oauth2 flow")

* ��A���û��򿪿ͻ����Ժ󣬿ͻ���Ҫ���û�������Ȩ��
* ��B���û�ͬ�����ͻ�����Ȩ��
* ��C���ͻ���ʹ����һ����õ���Ȩ������֤�������������ơ�
* ��D����֤�������Կͻ��˽�����֤�Ժ�ȷ������ͬ�ⷢ�����ơ�
* ��E���ͻ���ʹ�����ƣ�����Դ�����������ȡ��Դ��
* ��F����Դ������ȷ����������ͬ����ͻ��˿�����Դ��


## �ͻ��˵���Ȩģʽ

�ͻ��˱���õ��û�����Ȩ��authorization grant�������ܻ�����ƣ�access token����OAuth 2.0������������Ȩ��ʽ��

* ��Ȩ��ģʽ(Authorization Code)��������һ���������Ӧ��
* ��ģʽ(Implicit)�������ڴ���ҳ��Ӧ�ã����������Ƽ�ʹ�� PKCE ��Ϊ���
* ����ģʽ(Resource owner password credentials)
* �ͻ���ģʽ(Client credentials)


### ��Ȩ��ģʽ

��Ȩ��ģʽ��authorization code���ǹ��������������������ܵ���Ȩģʽ�������ص����ͨ���ͻ��˵ĺ�̨����������"�����ṩ��"����֤���������л�����

![oauth2 authorization code flow](images/oauth2_authorization_code_flow.png "oauth2 authorization code flow")

���Ĳ������£�

* ����΢���û����������΢����Ȩ��¼��ť�󣬶������Ὣ����ͨ��URL�ض���ķ�ʽ��ת��΢���û���Ȩ���棻
* ��ʱ΢���û�ʵ��������΢���Ͻ��������֤���붹�������޽����ˣ���һ��ǳ�����������ʹ������֧���ĳ�����
* �û�ʹ��΢�ſͻ���ɨ���ά����֤���������û��������΢�Ż���֤�û������Ϣ����ȷ�ԣ�����ȷ������Ϊ�û�ȷ����Ȩ΢�ŵ�¼����������ʱ��������һ����ʱƾ֤����Я����ƾ֤ͨ���û�������������ض���ض������ڵ�һ���ض���ʱЯ����callBackUrl��ַ��
* ֮���û��������Я����ʱƾ֤code���ʶ��������񣬶�������ͨ������ʱƾ֤�ٴε���΢����Ȩ�ӿڣ���ȡ��ʽ�ķ���ƾ��access_token��
* �ڶ�������ȡ��΢����Ȩ����ƾ��access_token�󣬴�ʱ�û�����Ȩ�����Ͼ�����ˣ�����������Ҫ����ֻ��ͨ����token�ٷ���΢���ṩ����ؽӿڣ���ȡ΢��������Ȩ�������û���Ϣ����ͷ���ǳƵȣ����ݴ����������û��߼����û���¼�Ự�߼���


### ��ģʽ

��ģʽ�Ƕ���Ȩ��ģʽ�ļ򻯣��������������ʹ�ýű�������JSʵ�ֵĿͻ����У������ص��ǲ�ͨ���ͻ���Ӧ�ó���ķ�����������ֱ���������������֤�������������ƣ������ˡ���Ȩ����ʱƾ֤��������衣�����еĲ��趼�����������ɣ����ƶԷ������ǿɼ��ģ��ҿͻ��˲���Ҫ��֤���������ʹ�ô�����Ȩ��ʽ��ʵ��΢�ŵ�¼�������Ĺ��̵Ļ����������£�

![oauth2 implicit grant flow](images/oauth2_implicit_grant_flow.png "oauth2 implicit grant flow")

������������п��Կ����ڵ�4���û������Ȩ����֤��������ֱ�ӷ�����access_token�������û�������ˣ�����û���ȷ�����Ȩ�룬Ȼ���ɿͻ��˵ĺ�˷���ȥͨ����Ȩ����ȥ��ȡaccess_token���ƣ��Ӷ�ʡȥ��һ����ת���裬����˽���Ч�ʡ�

�����������ַ�ʽ��������access_token����URLƬ���н��д��䣬��˿��ܻᵼ�·������Ʊ�����δ����Ȩ�ĵ�������ȡ�����԰�ȫ���ϲ�������ô��ǿ׳��


### ����ģʽ

������ģʽ�У��û���Ҫ��ͻ����ṩ�Լ����û��������룬�ͻ���ʹ����Щ��Ϣ�򡰷����ṩ�̡���Ҫ��Ȩ�����൱���ڶ�������ʹ��΢�ŵ�¼��������Ҫ�ڶ���������΢�ŵ��û��������룬Ȼ���ɶ�����ʹ�����ǵ�΢���û���������ȥ��΢�ŷ�������ȡ��Ȩ��Ϣ��

����ģʽһ�������û��Կͻ��˸߶����ε�����£���Ϊ��ȻЭ��涨�ͻ��˲��ô洢�û����룬����ʵ������һ�㲢�����ر��ǿ��Լ���������ڶ�����������΢�ŵ��û��������룬�������洢���洢���ǲ����Ǻ���������԰�ȫ���Ƿǳ��͵ġ����һ��������ǲ��ῼ��ʹ������ģʽ������Ȩ�ġ�

����΢�ŵ�¼�������Ĺ���ʹ���˴��ֿͻ�����Ȩģʽ�����������£�

![oauth2 password credentials flow](images/oauth2_password_credentials_flow.png "oauth2 password credentials flow")

����ģʽ�߼���Ҫ�ڷ�������ʵ�֣��޷�����ͻ���Ӧ�ó������Ȩ�������Ϣ�Ĵ洢������һ��������ǲ������������Ȩģʽ�ġ�


## �ͻ���ģʽ

�ͻ���ģʽ��ָ�ͻ������Լ������壬���������û������壬�򡰷����ṩ����������֤���ϸ��˵���ͻ���ģʽ��������OAuth2.0Э����Ҫ��������⡣������ģʽ�£��û�������Ҫ�Կͻ�����Ȩ���û�ֱ����ͻ���ע�ᣬ�ͻ������Լ�������Ҫ�󡰷����ṩ�̡��ṩ����

����������֮ǰ�������Ƚϻ������£����ֵ�ȫ�ܳ���ģʽ�����û���ȫ�ܳ�ע�ᣬȫ�ܳ����Լ�ƽ̨�����Ĺ����˻�ȥ������Ʒ�Ƶĵ�����

������������Ȼ��OAuth2.0Э���ж��������ֿͻ�����Ȩ��֤ģʽ������ʵ���ϴ󲿷�ʵ��Ӧ�ó�����ʹ�õĶ�����Ȩ�루authorization code����ģʽ����΢�ſ���ƽ̨��΢������ƽ̨��ʹ�õĻ���������Ȩ����֤ģʽ��


## KONG OAuth 2.0 Authentication

**1. ����·��**

·�ɵ�path����Ϊ/inner/apis/gateway/bin

**2. ��Ӳ��**

```CLI
 curl -X POST \
   --url http://127.0.0.1:8001/routes/route_test/plugins/ \
   --data "name=oauth2" \
   --data "config.scopes=email, phone, address" \
   --data "config.mandatory_scope=true" \
   --data "config.enable_authorization_code=true"
```

response����а��� provision_key , provision_key ����ʹ����webӦ�ú�Kong����Z֮���APIͨѶ,��ȷ��ͨѶ�İ�ȫ.

```CLI
{
    "route_id": "2c0c8c84-cd7c-40b7-c0b8-41202e5ee50b",
    "value": {
        "scopes": [
            "email",
            "phone",
            "address"
        ],
        "mandatory_scope": true,
        "provision_key": "2ef290c575cc46eec61947aa9f1e67d3",
        "hide_credentials": false,
        "enable_authorization_code": true,
        "token_expiration": 7200
    },
    "created_at": 1435783325000,
    "enabled": true,
    "name": "oauth2",
    "id": "656954bd-2130-428f-c25c-8ec47227dafa"
}
```

**3. ����������**

```CLI
curl -X POST \
  --url "http://127.0.0.1:8001/consumers/" \
  --data "username=thefosk"
```

**4. ����app**

Ϊ���Ŀ������˺����һ��app��app������Ϊ��Hello World App����

```CLI
curl -X POST \
  --url "http://127.0.0.1:8001/consumers/thefosk/oauth2/" \
  --data "name=Hello World App" \
  --data "redirect_uris[]=http://konghq.com/"
```

response �������:

```CLI
{
    "consumer_id": "a0977612-bd8c-4c6f-ccea-24743112847f",
    "client_id": "318f98be1453427bc2937fceab9811bd",        # Ĭ�����ɣ�������Ϊname
    "id": "7ce2f90c-3ec5-4d93-cd62-3d42eb6f9b64",
    "name": "Hello World App",
    "created_at": 1435783376000,
    "redirect_uri": "http://konghq.com/",
    "client_secret": "efbc9e1f2bcc4968c988ef5b839dd5a4"     # Ĭ�����ɣ�������Ϊpassword
}
```

**5. �ͻ��˻�ȡ Access token-�ͻ���ƾ֤ģʽ**

```CLI
curl -X POST \
  --url "https://127.0.0.1:8443/inner/apis/gateway/bin/oauth2/token" \
  --header "Host: mockbin.org" \
  --data "grant_type=password" \                                # ����basic Auth������������û��������룬��Ӧ��Ϊapp��client_id��client_secret
  --data "scope= xxxx " \                                       # auth2�������ʱ����
  --data "authenticated_useid= xxxx  " \
  --data "provision_key= 2ef290c575cc46eec61947aa9f1e67d3 " \   # auth2�����������ֵ
  --data "state = xxx " \
  --insecure

{
    "refresh_token": "N8YXZFNtx0onuuR7v465nVmnFN7vBKWk",
    "token_type": "bearer",
    "access_token": "njVmea9rlSbSUtZ2wDlHf62R7QKDgDhG",
    "expires_in": 7200
}
```

**6. �ͻ��˻�ȡ Access token-��Ȩ��ģʽ**

6.1 ��ȡCODE

```CLI
curl -X POST \
  --url "https://127.0.0.1:8443/inner/apis/gateway/bin/oauth2/authorize" \
  --header "Host: mockbin.org" \
  --data "client_id= xxxx " \                                   # ����appʱ��ȡ
  --data "scope= xxxx " \                                       # auth2�������ʱ����
  --data "authenticated_useid= xxxx  " \
  --data "provision_key= 2ef290c575cc46eec61947aa9f1e67d3 " \   # auth2�����������ֵ
  --data "response_type = code " \
  --data "redirect_uri = http://konghq.com/" \                  # ����appʱ����
  --data "state = xxx " \
  --insecure

{
	"redirect_uri = http://konghq.com/oauth2?code=xxxxxx&state=xxx
}
```

6.2 ʹ��CODE��ȡACCESSTOKEN

```CLI
curl -X POST \
  --url "https://127.0.0.1:8443/inner/apis/gateway/bin/oauth2/token" \
  --header "Host: mockbin.org" \
  --data "client_id= xxxx " \                                   # ����appʱ��ȡ
  --data "client_secret = xxxx " \                              # ����appʱ��ȡ
  --data "grant_type = authorization_code " \                  
  --data "scope= xxxx " \                                       # auth2�������ʱ����
  --data "code = xxxx  " \                                      # ��һ����ȡ
  --data "redirect_uri = http://konghq.com/" \                  # ����appʱ����
  --data "state = xxx " \
  --insecure
  
{
    "refresh_token": "N8YXZFNtx0onuuR7v465nVmnFN7vBKWk",
    "token_type": "bearer",
    "access_token": "njVmea9rlSbSUtZ2wDlHf62R7QKDgDhG",
    "expires_in": 7200
}
```

**7. ����API**

```CLI
curl -X GET \
  --url "http://127.0.0.1:8000/mock" \
  --header "Host: mockbin.org" \
  --header "Authorization: bearer njVmea9rlSbSUtZ2wDlHf62R7QKDgDhG"    # Я��access token����api
```

kong��ת�������ǣ����������header��

```CLI
    "x-consumer-id": "77e3f7ca-a969-48bb-a6d0-4a104ea7ad1e",
    "x-consumer-username": "thefosk",
    "x-authenticated-scope": "email address",
```
