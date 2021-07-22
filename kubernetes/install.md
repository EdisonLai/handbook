# kubernetes install guide

## 关闭防火墙
由于node间需要端口通信，在安全环境下，直接选择关闭防火墙
```text
systemctl disable firewalld
systemctl stop firewalld
```

## 禁用SELinux
```text
setenforce 0
```

## 增加yum源
```
cd /etc/yum.repos.d/  
vim kubernetes.repo  

[kubernetes]
name=Kubernetes Repository
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
```

## 安装各组件
```text
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
yum install docker  
systemctl enable docker && systemctl start docker  
systemctl enable kubelet && systemctl start kubelet
```

## 生成默认kubernetes配置文件
```text
kubeadm config print init-defaults > init.default.yaml
cp init.default.yaml init-config.yaml
```

## 修改docker源至国内镜像
```text
设置代理
echo '{"registry-mirrors":["https://registry.docker-cn.com"]}' > /etc/docker/daemon.json
```

