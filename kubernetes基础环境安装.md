### 1.基础环境
- 所有机器执行以下操作
```bash
#各个机器设置自己的域名
hostnamectl set-hostname xxxx

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#关闭swap
swapoff -a  
sed -ri 's/.*swap.*/#&/' /etc/fstab

#允许 iptables 检查桥接流量
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
### 2.安装kubelet、kubeadm、kubectl
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```
### 3.使用kubeadm引导集群

- 下载各个机器需要的镜像
```bash
sudo tee ./images.sh <<-'EOF'
#!/bin/bash
images=(
kube-apiserver:v1.20.9
kube-proxy:v1.20.9
kube-controller-manager:v1.20.9
kube-scheduler:v1.20.9
coredns:1.7.0
etcd:3.4.13-0
pause:3.2
)
for imageName in ${images[@]} ; do
docker pull registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/$imageName
done
EOF
   
chmod +x ./images.sh && ./images.sh
```

- 初始化主节点
```bash
#所有机器添加master域名映射，以下需要修改为自己的
echo "192.168.22.132  master" >> /etc/hosts



#主节点初始化
kubeadm init \
--apiserver-advertise-address=192.168.22.132 \
--control-plane-endpoint=master \
--image-repository registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images \
--kubernetes-version v1.20.9 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=172.16.0.0/16

#所有网络范围不重叠


```
#### k8s主节点初始化完成（根据主节点初始化完成信息进行操作）
```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join master:6443 --token 42c73t.afnylx9v0qm85hd1 \
    --discovery-token-ca-cert-hash sha256:53656428af3ff6e51b28b3fa12f396cd7945b38371df140d34e7ea5ea4c517ac \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join master:6443 --token 42c73t.afnylx9v0qm85hd1 \
    --discovery-token-ca-cert-hash sha256:53656428af3ff6e51b28b3fa12f396cd7945b38371df140d34e7ea5ea4c517ac 

```
```bash
#查看集群所有节点
kubectl get nodes

#根据配置文件，给集群创建资源
# kubectl apply -f xxxx.yaml

```
### 4.安装网络组件
```bash
curl https://docs.projectcalico.org/v3.20/manifests/calico.yaml -O

kubectl apply -f calico.yaml
```
#### 查看运行的应用
```bash
#查看集群部署了哪些应用？
docker ps   ===   kubectl get pods -A
# 运行中的应用在docker里面叫容器，在k8s里面叫Pod
kubectl get pods -A
```
> 创建新令牌
> kubeadm token create --print-join-command

### 5.加入node节点

- 将节点加入
```json

kubeadm join master:6443 --token 42c73t.afnylx9v0qm85hd1 \
    --discovery-token-ca-cert-hash sha256:53656428af3ff6e51b28b3fa12f396cd7945b38371df140d34e7ea5ea4c517ac 

```

- 验证集群
```bash
# 验证集群节点状态
kubectl get nodes
```
### 6、部署dashboard
#### 1、部署
> kubernetes官方提供的可视化界面
> [https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```
#### 2、设置访问端口
> 修改kubernetes-dashboard文件 将type: ClusterIP 改为 type: NodePort

```bash
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```
```bash
kubectl get svc -A |grep kubernetes-dashboard
## 找到端口，在安全组放行
```
访问： https://集群任意IP:端口      https://192.68.22.132:32269
#### Chrome浏览器访问https页面显示ERR_CERT_INVALID，且无法跳过继续访问

<img src=".\img\Snipaste_2023-01-02_09-24-39.png" alt="Snipaste_2023-01-02_09-24-39" style="zoom:60%;" />

> 在浏览器该界面上敲入这几个字符: **thisisunsafe **

#### 3、创建访问账号
```yaml
#创建访问账号，准备一个yaml文件； vi dash.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```
```bash
kubectl apply -f dash.yaml
```
#### 4、令牌访问
```bash
#获取访问令牌
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

```json
eyJhbGciOiJSUzI1NiIsImtpZCI6InlzVlhfNlNHTi1jMEt4U1BRYktwOVU5NWRPSV9WTGJFa3FobG5adGNUZVUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLW1jZHI0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI0NWNjNTlhNy1iZjgyLTQ0MDUtYmU2YS01ZjA5OTFlMmU3ODciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.Pvxsb0TV-jsp5VoaAqsuNqUHVzm_iJw4btrWZC3lw8M2OIrE9bphnWkgERz_JI6lzSyKdLKi9SN2zF-Vwx3ofPUbDDd9o8nDvKE26FFnfE8aeI0kspdeKc8xaAKRkPfXfN7vhD60TBA0IkMk3hLC41qH1fCGrD9u-VMebX9PR1_PaksTURfQPXlvghAVLJsFAXl2JaDTXH5hW1uL7NctO6Y5gKnePIyFbV2Y8gSHS_0oqcpXtTJVOfFbPc71_kZ6MS6QV0fJyx9gUmLYoOjaCZlGQce43-Wbr1jlX_9ku81cvM5jIKk6Wcxkh9_Y-23ggEvGT--Ky-jDHOQ-nppRiA
```
#### 5、界面


