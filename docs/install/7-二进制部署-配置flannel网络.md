
<!-- toc --> 

* * * * *
**flannel在每个node都要安装！**


## 1.创建flannel的csr请求json
```
[root@master temp]# cat flanneld-csr.json 
{
  "CN": "flanneld",
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


## 2.生成证书
```
[root@master temp]#  cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
   -ca-key=/etc/kubernetes/ssl/ca-key.pem \
   -config=/etc/kubernetes/ssl/ca-config.json \
   -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
2018/09/20 16:59:59 [INFO] generate received request
2018/09/20 16:59:59 [INFO] received CSR
2018/09/20 16:59:59 [INFO] generating key: rsa-2048
2018/09/20 16:59:59 [INFO] encoded CSR
2018/09/20 16:59:59 [INFO] signed certificate with serial number 219405115889238011180949290870567268914187527455
2018/09/20 16:59:59 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[root@master temp]# ll flann*
-rw-r--r-- 1 root root  997 Sep 20 16:59 flanneld.csr
-rw-r--r-- 1 root root  221 Sep 20 16:40 flanneld-csr.json
-rw------- 1 root root 1679 Sep 20 16:59 flanneld-key.pem
-rw-r--r-- 1 root root 1391 Sep 20 16:59 flanneld.pem
```


## 3.分发证书
```
[root@master temp]# cp flanneld*.pem ..
[root@master temp]# scp flanneld*.pem node1:/etc/kubernetes/ssl/
[root@master temp]# scp flanneld*.pem node2:/etc/kubernetes/ssl/
```


## 4.下载Flannel软件包
```
[root@master temp]# cd /usr/local/src
[root@master src]# tar zxf flannel-v0.10.0-linux-amd64.tar.gz 
[root@master src]# ls
cni-plugins-amd64-v0.7.1.tgz    flanneld                            kubernetes-client-linux-amd64.tar.gz  kubernetes.tar.gz
etcd-v3.3.9-linux-amd64         flannel-v0.10.0-linux-amd64.tar.gz  kubernetes-node-linux-amd64.tar.gz    mk-docker-opts.sh
etcd-v3.3.9-linux-amd64.tar.gz  kubernetes                          kubernetes-server-linux-amd64.tar.gz  README.md
```

## 5.分发二进制文件 和 脚本到各自节点
```
分发 二进制文件 flanneld    脚本mk-docker-opts.sh 
[root@master src]# cp flanneld mk-docker-opts.sh /etc/kubernetes/bin
[root@master src]# scp flanneld mk-docker-opts.sh node1:/etc/kubernetes/bin
[root@master src]# scp flanneld mk-docker-opts.sh node2:/etc/kubernetes/bin

分发 脚本remove-docker0.sh    
[root@master src]# cd /usr/local/src/kubernetes/cluster/centos/node/bin/
[root@master bin]# cp remove-docker0.sh /etc/kubernetes/bin/
[root@master bin]# scp remove-docker0.sh node1:/etc/kubernetes/bin/ 
[root@master bin]# scp remove-docker0.sh node2:/etc/kubernetes/bin/                                     
```

## 6.配置Flannel.cfg
```
[root@master cfg]# cat flannel.cfg
FLANNEL_ETCD="-etcd-endpoints=https://172.18.53.221:2379,https://172.18.53.223:2379,https://172.18.53.224:2379"
FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network"
FLANNEL_ETCD_CAFILE="--etcd-cafile=/etc/kubernetes/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="--etcd-certfile=/etc/kubernetes/ssl/flanneld.pem"
FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/etc/kubernetes/ssl/flanneld-key.pem"
```

## 7.分发Flannel.cfg
```
[root@master cfg]# scp flannel.cfg node1:/etc/kubernetes/cfg/
[root@master cfg]# scp flannel.cfg node2:/etc/kubernetes/cfg/

```


## 8.配置Flannel的service文件
```
[root@master cfg]# cat /usr/lib/systemd/system/flannel.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
Before=docker.service

[Service]
EnvironmentFile=-/etc/kubernetes/cfg/flannel.cfg
ExecStartPre=/etc/kubernetes/bin/remove-docker0.sh
ExecStart=/etc/kubernetes/bin/flanneld ${FLANNEL_ETCD} ${FLANNEL_ETCD_KEY} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
ExecStartPost=/etc/kubernetes/bin/mk-docker-opts.sh -d /run/flannel/docker

Type=notify

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
```

## 9.分发flannel的service配置文件
```
[root@master cfg]# scp /usr/lib/systemd/system/flannel.service node1:/usr/lib/systemd/system/flannel.service 
[root@master cfg]# scp /usr/lib/systemd/system/flannel.service node2:/usr/lib/systemd/system/flannel.service
```

## 10.Flannel CNI集成
### 10.1 配置cni的位置
```
[root@master cfg]# mkdir /etc/kubernetes/bin/cni
[root@master cfg]# tar zxf /usr/local/src/cni-plugins-amd64-v0.7.1.tgz -C /etc/kubernetes/bin/cni
[root@master cfg]# scp -r /etc/kubernetes/bin/cni/* node1:/etc/kubernetes/bin/cni/
[root@master cfg]# scp -r /etc/kubernetes/bin/cni/* node2:/etc/kubernetes/bin/cni/
```


### 10.2 创建Etcd的key
```
/etc/kubernetes/bin/etcdctl --ca-file /etc/kubernetes/ssl/ca.pem --cert-file /etc/kubernetes/ssl/flanneld.pem --key-file /etc/kubernetes/ssl/flanneld-key.pem \
--no-sync -C https://172.18.53.221:2379,https://172.18.53.223:2379,https://172.18.53.224:2379 \
mk /kubernetes/network/config '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}' >/dev/null 2>&1
```

```
[root@master bin]# systemctl daemon-reload
[root@master bin]# systemctl enable flannel
[root@master bin]# chmod +x /etc/kubernetes/bin/*
[root@master bin]# systemctl start flannel
```


### 10.3 查看服务状态
```
[root@master bin]# systemctl status flannel
● flannel.service - Flanneld overlay address etcd agent
   Loaded: loaded (/usr/lib/systemd/system/flannel.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2018-09-20 17:34:10 CST; 2min 0s ago
  Process: 2433 ExecStartPost=/etc/kubernetes/bin/mk-docker-opts.sh -d /run/flannel/docker (code=exited, status=0/SUCCESS)
  Process: 2417 ExecStartPre=/etc/kubernetes/bin/remove-docker0.sh (code=exited, status=0/SUCCESS)
 Main PID: 2424 (flanneld)
    Tasks: 8
   Memory: 9.1M
   CGroup: /system.slice/flannel.service
           └─2424 /etc/kubernetes/bin/flanneld -etcd-endpoints=https://172.18.53.221:2379,https://172.18.53.223:2379,https://172.18.53.224:2379 -etcd-prefix=/kubernetes/network --etcd-ca...

Sep 20 17:34:10 master flanneld[2424]: I0920 17:34:10.771155    2424 main.go:300] Wrote subnet file to /run/flannel/subnet.env
Sep 20 17:34:10 master flanneld[2424]: I0920 17:34:10.771166    2424 main.go:304] Running backend.
Sep 20 17:34:10 master flanneld[2424]: I0920 17:34:10.771448    2424 vxlan_network.go:60] watching for new subnet leases
Sep 20 17:34:10 master flanneld[2424]: I0920 17:34:10.778270    2424 main.go:396] Waiting for 22h59m59.991532601s to renew lease
Sep 20 17:34:10 master systemd[1]: Started Flanneld overlay address etcd agent.
Sep 20 17:34:10 master flanneld[2424]: I0920 17:34:10.821672    2424 iptables.go:115] Some iptables rules are missing; deleting and recreating rules
Sep 20 17:34:10 master flanneld[2424]: I0920 17:34:10.821692    2424 iptables.go:137] Deleting iptables rule: -s 10.2.0.0/16 -j ACCEPT
Sep 20 17:34:10 master flanneld[2424]: I0920 17:34:10.822675    2424 iptables.go:137] Deleting iptables rule: -d 10.2.0.0/16 -j ACCEPT
Sep 20 17:34:10 master flanneld[2424]: I0920 17:34:10.823622    2424 iptables.go:125] Adding iptables rule: -s 10.2.0.0/16 -j ACCEPT
Sep 20 17:34:10 master flanneld[2424]: I0920 17:34:10.825542    2424 iptables.go:125] Adding iptables rule: -d 10.2.0.0/16 -j ACCEPT
```

### 10.4 配置Docker使用Flannel
```
[root@node1 cni]# vim /usr/lib/systemd/system/docker.service
[Unit] #在Unit下面修改After和增加Requires
After=network-online.target firewalld.service flannel.service
Wants=network-online.target
Requires=flannel.service

[Service] #增加EnvironmentFile=-/run/flannel/docker
Type=notify
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_OPTS
```
Requires=flannel.service的意思是需要先把flannle启动之后，docker才能启动。
docker会使用环境变量中的配置文件，
```
[root@master src]# cat /run/flannel/docker
DOCKER_OPT_BIP="--bip=10.2.58.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_OPTS=" --bip=10.2.58.1/24 --ip-masq=true --mtu=1450"
```
所以docker启动的时候，用命令`ExecStart=/usr/bin/dockerd $DOCKER_OPTS`就可以使得docker用flannel的网络了(`bip`)


## 11. 将配置复制到节点2
```
root@node1 cni]# scp /usr/lib/systemd/system/docker.service node2:/usr/lib/systemd/system/docker.service
```

## 12. 在两个节点分别重启Docker
```
[root@node1 cni]# systemctl daemon-reload
[root@node1 cni]# systemctl restart docker
```