集群部件所需证书

| **CA&Key** | **etcd** | **kube-apiserver** | **kube-proxy** | **kubelet** | **kubectl** | **flanneld** |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| ca.pem | √ | √ | √ | √ | √ | √ |
| ca-key.pem |  |  |  |  |  |  |
| kubernetes.pem | √ | √ |  |  |  | √ |
| kubernetes-key.pem | √ | √ |  |  |  | √ |
| kube-proxy.pem |  |  | √ |  |  |  |
| kube-proxy-key.pem |  |  | √ |  |  |  |
| admin.pem |  |  |  |  | √ |  |
| admin-key.pem |  |  |  |  | √ |  |



1.安装CFSSL

```
# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/local/bin/cfssl
# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/local/bin/cfssljson
# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/local/bin//cfssl-certinfo 
# chmod +x /usr/local/bin/cfssl*
```

2.创建CA文件

```
# cd /usr/local/src && mkdir ssl && cd $_
```

2.1 创建CA配置文件

\#vim /usr/local/src/ssl/ca-config.json

```
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "87600h"
            }
        }
    }
}
```

字段说明:

* ca-config.json 可以定义读个 profiles,分别制定不同的国企时间,使用场景等参数;后续在签名时使用某个profile;
* signing:表示该证书可以用于签名其他证书;生成的ca.pem中CA=TRUE;
* server auth :表示client可以用该CA对server提供的证书进行验证;
* client auth :表示server可以用该CA对client提供的证书进行验证;

2.2 创建CA证书签名请求

\#vim /usr/local/src/ssl/ca-csr.json

```
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

* "CN": Common Name, kube-apiserver 从证书中提取该字段作为请求的用户名 \(User Name\);浏览器使用该字段验证网站是否合法;
* "O": Organization, kube-apiserver从证书中提取该字段作为请求用户所属的组\(Group\);

2.3 生成 CA 证书和私钥

```
[root@master ssl]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

![](/assets/cfssl_creat0001.png)

1. 创建 kubernetes 证书

3.1 创建 kubernetes 证书签名请求

vim /usr/local/src/ssl/kubernetes-csr.json

```
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "10.10.6.201",
      "10.10.4.12",
      "10.10.5.105",
      "internal-kubernetes-cluster-LB.cn-north-1.elb.amazonaws.com.cn",
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

* 如果hosts字段不为空,则需要制定授权证书的IP或域名列表,由于该证书后续将被etcd集群和kubernetes master集群所使用,
* 所以上面指定了etcd集群,master集群的主机域名,kubernetes服务的服务 IP 
* 一般是kue-apiserver指定的service-cluster-ip-range网段的第一个IP，如 10.254.0.1,kubernetes一定要加,不然后面会遇到很多坑,
* 另外本例的LB并没有使用公网的DNS，建议使用公网的DNS
* 此处的IP仅为master节点的IP，复用etcd数据库地址，添加node时无需额外生成证书

3.2 生成 kubernetes 证书和私钥

```
[root@master ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
```

![](/assets/kubernet0003.png)

4.创建 admin 证书

4.1 创建 admin 证书签名请求

vim /usr/local/src/ssl/admin-csr.json

```
{
    "CN": "admin",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "system:masters",
            "OU": "System"
        }
    ]
}
```

* 后续kube-apiserver 使用 RBAC对客户端\(如 kubelet、kube-proxy、Pod\)请求进行授权；
* kube-apiserver 预定义了一些 RBAC使用的 RoleBindings，如 cluster-admin 将 Group system:masters与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver的所有 API的权限；
* OU指定该证书的 Group为 system:masters，kubelet 使用该证书访问 kube-apiserver时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限；

4.1 生成 admin 证书和私钥

```
[root@master ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

![](/assets/admin003.png)

5.创建kube-proxy 证书

5.1 创建 kube-proxy 证书签名请求

vim /usr/local/src/ssl/kube-proxy-csr.json

```
{
    "CN": "system:kube-proxy",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

* CN指定该证书的 User为 system:kube-proxy；
* kube-apiserver预定义的RoleBinding cluster-admin 将User system:kube-proxy 与 Role system:node-proxier绑定，该 Role 授予了调用 kube-apiserver Proxy相关 API 的权限；

5.2 生成 kube-proxy 客户端证书和私钥

```
#cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

![](/assets/kube-proxycre0002.png)

6.校验证书，以kubernetes证书为例

6.1 使用openssl 命令

```
[root@master ssl]# openssl x509  -noout -text -in  kubernetes.pem
```

![](/assets/key0004.png)

* 确认Issuer字段的内容和 ca-csr.json一致；
* 确认 Subject字段的内容和 kubernetes-csr.json 一致；
* 确认 X509v3 Subject Alternative Name字段的内容和 kubernetes-csr.json 一致；
* 确认X509v3 Key Usage、Extended Key Usage 字段的内容和 ca-config.json 中 kubernetes profile一致；

6.2 使用`cfssl-certinfo`命令

```
[root@master ssl]# cfssl-certinfo -cert kubernetes.pem
```

![](/assets/cfssl_00045.png)



7.分发证书

将生成的证书和秘钥文件（后缀名为.pem）拷贝到所有机器的`/etc/kubernetes/ssl`目录下备用；

```
# for i in k8s-1 k8s-2 k8s-3 k8s-4 ;do scp *pem root@$i:/etc/kubernetes/ssl/ ;done
```



