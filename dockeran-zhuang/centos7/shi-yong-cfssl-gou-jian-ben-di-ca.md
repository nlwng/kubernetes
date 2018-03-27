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

3 创建CA配置文件

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

4.创建CA证书签名请求

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

5.生成 CA 证书和私钥

```

```



