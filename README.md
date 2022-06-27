# Install a highly available k8s cluster using Ansible

```sh
# ⚠️ 参考手动安装，把集群第一台主机作为 Ansible 控制机

# 安装 ansible
pip install ansible==5.8.0

# 获取安装脚本
git clone https://github.com/princeofdatamining/k8ansible/ ~/k8s-ansible

cd ~/k8s-ansible

# 为方便后续引用，定义集群 inventory 目录名变量(减少替换次数)
CLUSTER_INVENTORY=demo # TODO: change it

# 复制 ansible 配置文件，并设置默认 inventory 目录
cat ansible.cfg.sample | sed "s|^inventory.*|inventory = inventories/${CLUSTER_INVENTORY} |" | tee ansible.cfg

# 为区分集群，每个集群单独一个目录(可复制示例目录)
cp -r inventories/sample inventories/${CLUSTER_INVENTORY}

# 列出需要修改的配置项
grep TODO -r inventories/${CLUSTER_INVENTORY}/*

# 编辑主机清单文件
vi inventories/${CLUSTER_INVENTORY}/hosts.yml

# 编辑变量清单文件
vi inventories/${CLUSTER_INVENTORY}/group_vars/all/vars.yml

# 0x00
# 从 hosts 和 vars 推导出复杂变量，以直接使用(例如 etcd 集群配置)
ansible-playbook 00-gen-vars.yml

# 0x01 - 环境准备
# 1. 停用 swap
# 2. 停用 selinux
# 3. 停用防火墙
# 4. 设置时区，启用时间同步
# 5. 设置 /etc/hosts
# 6. export KUBECONFIG
# 7. yum install ...
# 8. export ETCD_ 环境变量
# 9. 生成 kubeadm init phases 需要的 config 文件
ansible-playbook 01-environ.yml

# 0x02 - 组件下载
# 1. k8s 组件下载安装
# 2. 容器运行时配置(Container Runtime)
# 3. CNI 组件下载安装
ansible-playbook 02-download.yml

# 0x03 - load balancer(floating ip)
# Control plane SLB & External traffic SLB
ansible-playbook 03-load-balancer.yml

# 0x04 - kubelet.service
# Configure, enable and start kubelet systemd service
ansible-playbook 04-kubelet.yml

# OPTIONAL: 容器镜像下载(检查代理是否生效)
# ⚠️ 默认不更改各容器镜像地址，推荐配置仓库代理以保正常拉取所有容器镜像
# v1.24.0
# 控制平面组件镜像
crictl pull k8s.gcr.io/kube-apiserver:v1.24.0
crictl pull k8s.gcr.io/kube-controller-manager:v1.24.0
crictl pull k8s.gcr.io/kube-scheduler:v1.24.0
crictl pull k8s.gcr.io/etcd:3.5.3-0
# 工作节点组件镜像
crictl pull k8s.gcr.io/kube-proxy:v1.24.0
crictl pull k8s.gcr.io/pause:3.7
# 插件镜像
crictl pull k8s.gcr.io/coredns/coredns:v1.8.6
crictl pull docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.0
crictl pull docker.io/rancher/mirrored-flannelcni-flannel:v0.18.1
crictl pull quay.io/tigera/operator:v1.27.1
crictl pull docker.io/calico/apiserver:v3.23.1
crictl pull docker.io/calico/cni:v3.23.1
crictl pull docker.io/calico/kube-controllers:v3.23.1
crictl pull docker.io/calico/node:v3.17.1
crictl pull docker.io/calico/node:v3.23.1
crictl pull docker.io/calico/pod2daemon-flexvol:v3.23.1
crictl pull docker.io/calico/typha:v3.23.1
crictl pull k8s.gcr.io/metrics-server/metrics-server:v0.6.1

# 0x0C - HA etcd cluster
# 部署独立的 etcd 高可用集群
ansible-playbook 12-ha-etcd.yml

# 0x0D - HA k8s cluster
# 部署 k8s 高可用集群
ansible-playbook 13-ha-cluster.yml

# 0x14 - CNI
# ⚠️ cni-plugins-linux-amd64-{{ version.cni }}.tgz 与 插件容器镜像的 CNI 协议版本要一致
# 安装 CNI 插件
ansible-playbook 30-cni-flannel.yml
### OR calico: overlay #
# ansible-playbook 30-cni-calico-overlay.yml
### OR calico: BGP #
# ansible-playbook 30-cni-calico-bgp.yml # 默认是 node-to-node mesh
# ansible-playbook 30-cni-calico-bgp.yml -t bgp-reflector # calico 组件运行后，指定 reflector nodes
# ansible-playbook 30-cni-calico-bgp.yml -t bgp-no-mesh # 取消 node-to-node mesh

```
