---
title: "Kubernetes集群证书过期解决办法"
date: 2020-05-10T03:19:48+08:00
draft: false
tags: ["证书过期"]
series: ["K8S运维"]
categories: ["K8S"]
---



### 一、简述集群异常情况

> 本地测试环境配置：
>
> master&node 01:  192.168.188.150
>
> master&node 02:  192.168.188.160
>
> master&node 03:  192.168.188.161



今天查看k8s集群情况，发现所有node都处于NotReady状态。

```shell
[root@localhost ~]# kubectl get node
NAME              STATUS     AGE       VERSION
192.168.188.150   NotReady   1y        v1.6.0
192.168.188.160   NotReady   1y        v1.6.0
192.168.188.161   NotReady   1y        v1.6.0
```



登陆master 查看api-server日志

```shell
journalctl -xefu kube-apiserver

5月 10 17:10:47 localhost.localdomain kube-apiserver[2194]: E0510 17:10:47.800829    2194 authentication.go:58] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
5月 10 17:10:47 localhost.localdomain kube-apiserver[2194]: E0510 17:10:47.904426    2194 authentication.go:58] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
5月 10 17:10:47 localhost.localdomain kube-apiserver[2194]: E0510 17:10:47.997393    2194 authentication.go:58] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
```

错误日志显示k8s证书过期：

Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid



登陆node01节点，查看kubelet日志

```shell
[root@localhost ~]# journalctl -u kubelet |tail
5月 10 17:15:14 localhost.localdomain kubelet[3374]: E0510 17:15:14.597154    3374 reflector.go:190] k8s.io/kubernetes/pkg/kubelet/kubelet.go:382: Failed to list *v1.Service: the server has asked for the client to provide credentials (get services)
5月 10 17:15:15 localhost.localdomain kubelet[3374]: E0510 17:15:15.110341    3374 reflector.go:190] k8s.io/kubernetes/pkg/kubelet/kubelet.go:390: Failed to list *v1.Node: the server has asked for the client to provide credentials (get nodes)
5月 10 17:15:15 localhost.localdomain kubelet[3374]: E0510 17:15:15.187072    3374 reflector.go:190] k8s.io/kubernetes/pkg/kubelet/config/apiserver.go:46: Failed to list *v1.Pod: the server has asked for the client to provide credentials (get pods)
```

日志显示server端提供证书

the server has asked for the client to provide credentials



问题初步定位：k8s证书异常



### 二、检查k8s证书是否过期

1、登陆master检查apiserver证书情况

```shell
[root@localhost ~]# ps -ef |grep kube-apiserver |grep -v grep
root       2194      1  2 13:54 ?        00:05:12 /usr/local/bin/kube-apiserver --logtostderr=true --v=0 --etcd-servers=https://192.168.188.150:2379,https://192.168.188.160:2379,https://192.168.188.161:2379 --advertise-address=192.168.188.150 --bind-address=192.168.188.150 --insecure-bind-address=192.168.188.150 --allow-privileged=true --service-cluster-ip-range=10.254.0.0/16 --admission-control=ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota --authorization-mode=RBAC --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true --experimental-bootstrap-token-auth --token-auth-file=/etc/kubernetes/token.csv --service-node-port-range=30000-32767 --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem --etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem --enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/lib/audit.log --event-ttl=1
```

从kube-apiserver的启动参数可以看到当前证书配置的位置。附一些参数的说明

`--tls-cert-file`  

> If set, this certificate authority will used for secure access from Admission Controllers. This must be a valid PEM-encoded CA bundle. Alternatively, the certificate authority can be appended to the certificate provided by --tls-cert-file.

`--tls-private-key-file`

> File containing the default x509 private key matching --tls-cert-file.

`--client-ca-file`

> If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate.

`--service-account-key-file`

> File containing PEM-encoded x509 RSA or ECDSA private or public keys, used to verify ServiceAccount tokens. If unspecified, --tls-private-key-file is used. The specified file can contain multiple keys, and the flag can be specified multiple times with different files.

k8s证书列表说明：https://www.jianshu.com/p/549ab7a059b0



我的集群k8s证书都存放在/etc/kubernetes/ssl/

```shell
[root@localhost ~]# cd /etc/kubernetes/ssl/
[root@localhost ssl]# ls
admin-key.pem  ca-key.pem  kubelet-client.crt  kubelet.crt  kube-proxy-key.pem  kubernetes-key.pem
admin.pem      ca.pem      kubelet-client.key  kubelet.key  kube-proxy.pem      kubernetes.pem
```



#### kubelet证书介绍

##### token.csv

该文件为一个用户的描述文件，基本格式为 Token,用户名,UID,用户组；这个文件在 apiserver 启动时被 apiserver 加载，然后就相当于在集群内创建了一个这个用户；接下来就可以用 RBAC 给他授权；持有这个用户 Token 的组件访问 apiserver 的时候，apiserver 根据 RBAC 定义的该用户应当具有的权限来处理相应请求

##### bootstarp.kubeconfig

该文件中内置了 token.csv 中用户的 Token，以及 apiserver CA 证书；kubelet 首次启动会加载此文件，使用 apiserver CA 证书建立与 apiserver 的 TLS 通讯，使用其中的用户 Token 作为身份标识像 apiserver 发起 CSR 请求

##### kubelet-client.crt

该文件在 kubelet 完成 TLS bootstrapping 后生成，此证书是由 controller manager 签署的，此后 kubelet 将会加载该证书，用于与 apiserver 建立 TLS 通讯，同时使用该证书的 CN 字段作为用户名，O 字段作为用户组向 apiserver 发起其他请求

##### kubelet.crt

该文件在 kubelet 完成 TLS bootstrapping 后并且没有配置 --feature-gates=RotateKubeletServerCertificate=true 时才会生成；这种情况下该文件为一个独立于 apiserver CA 的自签 CA 证书，有效期为 1 年；被用作 kubelet 10250 api 端口

##### kubelet-server.crt

该文件在 kubelet 完成 TLS bootstrapping 后并且配置了 --feature-gates=RotateKubeletServerCertificate=true 时才会生成；这种情况下该证书由 apiserver CA 签署，默认有效期同样是 1 年，被用作 kubelet 10250 api 端口鉴权

##### kubelet-client-current.pem

这是一个软连接文件，当 kubelet 配置了 --feature-gates=RotateKubeletClientCertificate=true选项后，会在证书总有效期的 70%~90% 的时间内发起续期请求，请求被批准后会生成一个 kubelet-client-时间戳.pem；kubelet-client-current.pem 文件则始终软连接到最新的真实证书文件，除首次启动外，kubelet 一直会使用这个证书同 apiserver 通讯

##### kubelet-server-current.pem

同样是一个软连接文件，当 kubelet 配置了 --feature-gates=RotateKubeletServerCertificate=true 选项后，会在证书总有效期的 70%~90% 的时间内发起续期请求，请求被批准后会生成一个 kubelet-server-时间戳.pem；kubelet-server-current.pem 文件则始终软连接到最新的真实证书文件，该文件将会一直被用于 kubelet 10250 api 端口鉴权



查看证书是否过期：

```shell
[root@localhost ssl]# openssl x509 -in /etc/kubernetes/ssl/kubelet.crt -noout -text |grep ' Not '
            Not Before: Jan  6 10:38:32 2019 GMT
            Not After : Jan  6 10:38:32 2020 GMT
            
[root@localhost ssl]# openssl x509 -in /etc/kubernetes/ssl/kubernetes.pem -noout -text |grep ' Not '
            Not Before: Dec 15 12:45:00 2018 GMT
            Not After : Dec 12 12:45:00 2028 GMT

[root@localhost ssl]# openssl x509 -in /etc/kubernetes/ssl/ca.pem -noout -text |grep ' Not '
            Not Before: Dec 15 10:46:00 2018 GMT
            Not After : Dec 12 10:46:00 2028 GMT
[root@localhost ssl]# openssl x509 -in /etc/kubernetes/ssl/kube-proxy.pem -noout -text |grep ' Not '
            Not Before: Dec 15 13:59:00 2018 GMT
            Not After : Dec 12 13:59:00 2028 GMT
[root@localhost ssl]# openssl x509 -in /etc/kubernetes/ssl/admin.pem -noout -text |grep ' Not '
            Not Before: Dec 15 13:48:00 2018 GMT
            Not After : Dec 12 13:48:00 2028 GMT

```

`Not After` 表示过期时间。

可以看到，`kubelet.crt`的证书只有一年有效期，且已经过期。
其余证书有效期10年，还未过期。（那么问题来了，10年的证书过期后的更换方法）

集群分为两种证书：
1.用于集群 Master、Etcd等通信的证书。
2.用于集群 Kubelet 组件证书。

我们在搭建 Kubernetes 集群时，一般只声明用于集群 Master、Etcd等通信的证书 为 10年 或者 更久，但未声明集群 Kubelet 组件证书 ，Kubelet 组件证书 默认有效期为1年。集群运行1年以后就会导致报 certificate has expired or is not yet valid 错误，导致集群 Node不能于集群 Master正常通信，node显示是NotReady状态。

### 三、解决方案

#### 1、临时解决方案

1.1 删除相关过期证书。

删除之前，先备份是个好习惯

```shell
cp -R /etc/kubernetes/ /etc/kubernetes.bak
```

```
rm  /etc/kubernetes/kubelet.kubeconfig
rm  /etc/kubernetes/ssl/kubelet.*
```

在证书过期node节点删除kubelet相关证书文件及配置文件然后重启kubelet,kubelet会向apiserver发起一个csr。



1.2 重启node节点上kube-proxy 和 kubelet 服务

```shell
systemctl daemon-reload
systemctl restart kubelet
systemctl restart kube-proxy

systemctl status kubelet
```



1.3 删除NotReady节点重新加入k8s集群

```
kubectl  delete node 192.168.188.150 192.168.188.160 192.168.188.161
```



1.4 查看未授权的CSR请求，并授权

``` shell
[root@localhost ssl]# kubectl get csr |grep Pending
csr-75b4g   2m        kubelet-bootstrap   Pending
csr-p3r0s   3m        kubelet-bootstrap   Pending
csr-pn8kr   3m        kubelet-bootstrap   Pending
```

``` shell
[root@localhost ssl]# kubectl certificate approve csr-75b4g
certificatesigningrequest "csr-75b4g" approved
[root@localhost ssl]# kubectl certificate approve csr-p3r0s 
certificatesigningrequest "csr-p3r0s" approved
[root@localhost ssl]# kubectl certificate approve csr-pn8kr 
certificatesigningrequest "csr-pn8kr" approved
```



1.5 验证，所有node已处于Ready状态

```shell
[root@localhost ssl]# kubectl get node
NAME              STATUS    AGE       VERSION
192.168.188.150   Ready     40s       v1.6.0
192.168.188.160   Ready     47s       v1.6.0
192.168.188.161   Ready     32s       v1.6.0

```



#### 2、【永久解决方案】设置证书轮换。需要k8s 1.8版本以上

https://kubernetes.io/zh/docs/tasks/tls/certificate-rotation/

https://blog.csdn.net/feifei3851/article/details/88390425

由于我的测试环境k8s版本是1.6，后续升级k8s版本后再验证该功能。



k8s集群搭建资料：https://jimmysong.io/kubernetes-handbook/practice/create-tls-and-secret-key.html


参考资料：https://blog.csdn.net/qq_25934401/article/details/105391819
