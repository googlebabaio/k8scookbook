为了使用这个方案，api－server需要用－token－auth－file＝<PATH_TO_TOKEN_FILE>选项来开启。TOKEN_FILE是个csv文件，每个用户入口都有下列格式：token，user，userid，group。

Group的名字是随意的。

令牌文件的例子：
```
[root@master temp]# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
cdf6d36d697dae799728ac52531a43a8
[root@master temp]# vim /etc/kubernetes/ssl/bootstrap-token.csv
[root@master temp]# cat /etc/kubernetes/ssl/bootstrap-token.csv
cdf6d36d697dae799728ac52531a43a8,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
[root@master temp]# 
```
基于令牌的身份验证面临的挑战就是，令牌是无期限的，而且对令牌清单做任何的修改都需要重新启动api－server。