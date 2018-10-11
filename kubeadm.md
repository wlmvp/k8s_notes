####kubeadm###############


#0.1 关掉 selinux
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
#0.2 关掉防火墙
systemctl stop firewalld
systemctl disable firewalld
#0.3 关闭 swap
swapoff -a 
sed -i 's/.*swap.*/#&/' /etc/fstab
#0.4 配置转发参数
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
#0.5 设置国内 yum 源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
#0.6 安装一些必备的工具
yum install -y epel-release 
yum install -y net-tools wget vim  ntpdate


1. 安装 kubeadm 必须的软件，在所有节点上运行
1.1 安装Docker
yum install -y docker
systemctl enable docker && systemctl start docker
#设置系统服务，如果不设置后面 kubeadm init 的时候会有 warning
systemctl enable docker.service
#如果想要用二进制方法安装最新版本的Docker，可以参考我之前的文章在Redhat 7.3中采用离线方式安装Docker

#1.2 安装kubeadm、kubectl、kubelet
yum install -y kubelet kubeadm kubectl kubernetes-cni

#如下操作在所有节点操作
# 配置kubelet使用国内pause镜像
# 配置kubelet的cgroups
# 获取docker的cgroups
DOCKER_CGROUPS=$(docker info | grep 'Cgroup' | cut -d' ' -f3)
echo $DOCKER_CGROUPS
cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=$DOCKER_CGROUPS --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
EOF

# 启动
systemctl daemon-reload
systemctl enable kubelet && systemctl restart kubelet
#这一步之后kubelet还不能正常运行，还处于下面的状态。
#The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.



cat >kubeadm.yaml<<EOF
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
api:
  advertiseAddress: "192.168.119.151"
networking:
  podSubnet: "10.244.0.0/16"
kubernetesVersion: "v1.12.1"
imageRepository: "registry.cn-hangzhou.aliyuncs.com/google_containers"
EOF

kubeadm config images pull --config=kubeadm.yaml

##
cat >imagespull.sh<<EOF
#!/bin/bash
images=(kube-proxy-amd64:v1.12.1 kube-scheduler-amd64:v1.12.1 kube-controller-manager-amd64:v1.12.1 kube-apiserver-amd64:v1.12.1
etcd-amd64:3.2.24 coredns:1.2.2 pause-amd64:3.1 kubernetes-dashboard-amd64:v1.8.3 k8s-dns-sidecar-amd64:1.14.9 k8s-dns-kube-dns-amd64:1.14.9
k8s-dns-dnsmasq-nanny-amd64:1.14.9 )
for imageName in ${images[@]} ; do
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
  docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
docker tag da86e6ba6ca1 k8s.gcr.io/pause:3.1
EOF

kubeadm init --config=kubeadm.yaml
#####if error: kubeadm reset

export KUBECONFIG=/etc/kubernetes/admin.conf


kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml

containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=enp0s3            #指定内网网卡
		
		
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - operator: Exists



默认token的有效期为24小时，当过期之后，该token就不可用了。
kubeadm token create //如过期，创建token

[root@kube-node1 home]# kubeadm token list
TOKEN TTL EXPIRES USAGES DESCRIPTION EXTRA GROUPS
1eg5gp.zmys0lirp7bikh7l 8h 2018-07-31T17:07:35+08:00 authentication,signing <none> system:bootstrappers:kubeadm:default-node-token

获取ca证书sha256编码hash值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
5443ce1592bb287ba362cedd3128c261c108c2c44fd48b5a60b90ee4e8460a3f

节点加入集群
kubeadm join --token 1eg5gp.zmys0lirp7bikh7l --discovery-token-ca-cert-hash sha256:5443ce1592bb287ba362cedd3128c261c108c2c44fd48b5a60b90ee4e8460a3f 192.168.0.93:6443 --skip-preflight-checks