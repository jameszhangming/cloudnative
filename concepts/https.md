# ��������

## ��Կ��˽Կ

**˽Կ�����㷨 == �ԳƼ����㷨**

�����㷨������Կ�ͼ�����Կ����ͬ�ģ�������Ϊͬһ����Կ�����ڼ��������ڽ��ܣ�������Կ���ܹ�����

�����ĶԳƼ����㷨�У�AES��DES��3DES��

**��Կ�㷨 == �ǶԳƼ����㷨**

1. ��Կ˽Կ�ɶԳ���
2. ��Կ���ܵ�����ֻ�ж�Ӧ��˽Կ�ܽ⣬˽Կ���ܵ�����ֻ�ж�Ӧ�Ĺ�Կ�ܽ�

�����ķǶԳƼ����㷨�У�RSA��DSA��ECC��

�ԳƼ����㷨�ǷǶԳƼ������ܵ�1500����

## ����ժҪ

�����ߵ���Ϣ����hash�㷨����õ���digestժҪ������ժҪ��HTTPS��ȷ�����ݵ������Ժͷ��۸ĵĸ���ԭ��

## ����ǩ��
�Դ�����������ժҪ��ʹ��˽Կ���м��ܵĹ��̾�������������ǩ���Ĺ��̣�����ǩ��ֻ����֤���ݵ������ԣ����ݱ����Ƿ���ܲ���������ǩ���Ŀ��Ʒ�Χ��

����ǩ�����������ã�

1. ȷ����Ϣ���ɷ��ͷ�ǩ������������
2. ȷ����Ϣ��������

# TLS ֤�����ɺ�ʹ��

**1. ���� CA ˽Կ�Լ�֤��**

```CLI
# ���� CA ˽Կ ca.key.pem
openssl genrsa -out ca.key.pem 4096

# ���� CA ֤�� ca.cert.pem
openssl req -key ca.key.pem -new -x509 -days 7300 -sha256 -out ca.cert.pem -subj /CN=ca -extensions v3_ca
```

**2. ���� Server ˽Կ��֤��**

```CLI
# ���� server ˽Կ server.key.pem�������ͬCA˽Կ
openssl genrsa -out server.key.pem 2048

# ���� server ֤�������ļ� server.cnf
cat <<EOF >  server.cnf
[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name
req_extensions     = req_ext
[ req_distinguished_name ]
[ req_ext ]
basicConstraints = CA:FALSE
subjectAltName = @alt_names
[ alt_names ]
DNS.1   = localhost
DNS.2   = *test-server-001
IP.1    = 127.0.0.1
EOF
# ���� server ֤�������ļ� server.csr.pem
openssl req -key server.key.pem -new -sha256 -out server.csr.pem -subj /CN=server -config server.cnf

# ���� server ֤�� server.cert.pem����Կʹ��CAǩ��
openssl x509 -req -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -in server.csr.pem -out server.cert.pem -days 365 -extensions req_ext -extfile server.cnf
```

**3. ���� Client ˽Կ��֤��**

```CLI
# ���� client ˽Կ client.key.pem
openssl genrsa -out client.key.pem 2048
# ���� client ֤�������ļ� client.csr.pem
openssl req -key client.key.pem -new -sha256 -out client.csr.pem -subj /CN=client
# ���� client ֤�� client.cert.pem����Կʹ��CAǩ��
openssl x509 -req -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -in client.csr.pem -out client.cert.pem -days 365
```

# HTTPSԭ��

����HTTP Э��ͨ�ŵĲ���ȫ�ԣ���������Ϊ�˷�ֹ��Ϣ�ڴ���������⵽й©���ߴ۸ģ���������Դ���ͨ�����м��ܵķ�ʽhttps��https ��һ�ּ��ܵĳ��ı�����Э�飬����HTTP ��Э��������ڶ����ݴ���Ĺ����У�https ������������ȫ���ܡ�����http Э�����httpsЭ�鶼�Ǵ���TCP �����֮�ϣ�ͬʱ����Э������һ���ֲ�Ľṹ��������tcp Э���֮��������һ��SSL��Secure Socket Layer����ȫ�㣩����TLS��Transport Layer Security�� ��ȫ�㴫��Э�����ʹ�����ڹ������ͨ����

![https arch](images/https arch.png "https arch")

## HTTPS��������

![https flow](images/https flow.png "https flow")

����˵����

1. �ͻ��˷��ʣ�https://www.z.com���ͻ�����Ҫ��������ṩ������Ϣ��
  * ֧�ֵ�Э��汾������TLS 1.0�档
  * һ���ͻ������ɵ���������Ժ���������"�Ի���Կ"��
  * ֧�ֵļ��ܷ���������RSA��Կ���ܡ�
  * ֧�ֵ�ѹ��������
2. https�ļ����˿���443�˿ڣ�����nginx��������nginx.conf �����������ã��������ã�

```CLI
server
{
  listen 443;   # https ��������?443�˿�
  server_name www.z.com;

  ssl on;
  ssl_session_cache shared:SSL:10m;
  ssl_session_timeout 10m;

  ssl_certificate /etc/nginx/ssl_key/z.crt.crt;  # ��Կ
  ssl_certificate_key /etc/nginx/ssl_key/z.crt.key;  # ˽Կ
}
```
nginx ������������õ�ssl_certificate ��·��?/etc/nginx/ssl_key/z.crt.crt �ҵ�www.z.com ����֤��Ĺ�Կ������Կ���ظ��ͻ��ˣ��Լ�������Ϣ��
  * ȷ��ʹ�õļ���ͨ��Э��汾������TLS 1.0�汾�����������������֧�ֵİ汾��һ�£��������رռ���ͨ�š�
  * һ�����������ɵ���������Ժ���������"�Ի���Կ"��
  * ȷ��ʹ�õļ��ܷ���������RSA��Կ���ܡ�
3. �ͻ��˽��յ���Կ�󣬻�Թ�Կ���ݽ��н�����У�飬֤�����֤�������£�
  * CA������ǩ��֤���ʱ�򣬶���ʹ���Լ���˽Կ��֤�����ǩ�����������ʹ�õ��ǹ����֤�飬��ô���п��ܣ��䷢���֤���CA�����Ĺ�Կ�Ѿ�Ԥ���ڲ���ϵͳ�С�����������Ϳ���ʹ��CA�����Ĺ�Կ�Է�������֤�������ǩ����ǩ֮��õ�����CA����ʹ��sha256�õ���֤��ժҪ���ͻ��˾ͻ�Է��������͹�����֤��ʹ��sha256���й�ϣ����õ�һ��ժҪ��Ȼ��Ա�֮ǰ��CA�ó�����ժҪ���Ϳ���֪�����֤���ǲ�����ȷ�ģ��Ƿ��޸Ĺ���
4. У��ͨ���󣬿ͻ��˻��������һ��randomKey�������Կ��
5. �ͻ�����ʹ��z.crt.crt�еĹ�Կ����?randomKey ���м��ܣ������ܺ�����ݷ��͸������
  * ����ı�֪ͨ����ʾ������Ϣ������˫���̶��ļ��ܷ�������Կ���͡�
  * �ͻ������ֽ���֪ͨ����ʾ�ͻ��˵����ֽ׶��Ѿ���������һ��ͬʱҲ��ǰ�淢�͵��������ݵ�hashֵ��������������У��
6. ����˽��ձ��ĺ�ʹ��z.crt.key �е�˽Կ���Լ��ܵ����ݵĽ��н��ܣ�Ȼ��õ�?randomKey?
7. ��ʱ������˻Ὣ������������Ӧ�����ʹ��?randomKey? ���м��ܣ����ؿͻ���
  * ����ı�֪ͨ����ʾ������Ϣ������˫���̶��ļ��ܷ�������Կ���͡�
  * ���������ֽ���֪ͨ����ʾ�����������ֽ׶��Ѿ���������һ��ͬʱҲ��ǰ�淢�͵��������ݵ�hashֵ���������ͻ���У�顣
8. �ͻ��˶Է���˵ļ��ܱ���ʹ��randomKey?���ܣ�Ȼ��Ϳ���չʾ���������