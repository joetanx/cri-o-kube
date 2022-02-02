> Setup guide for single-node Kubernetes cluster with Flannel networking on RHEL8
# Install CRI-O
```console
OS=CentOS_8_Stream
VERSION=1.23
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
yum -y install cri-o
systemctl enable --now crio
```
- Configure required kernel modules and tunables
- Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#letting-iptables-see-bridged-traffic
```console
cat <<EOF >> /etc/modules-load.d/k8s.conf
br_netfilter
EOF
modprobe br_netfilter
cat <<EOF >> /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
# Install kubeadm, kubelet and kubectl
- Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl
- Note:
  - CRI-O uses the systemd cgroup driver per default. Ref: https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-driver
  - There are several ways to configure the cgroup driver, this guide configures it in `/etc/default/kubelet`. Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#the-kubelet-drop-in-file-for-systemd
```console
cat <<EOF >> /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
yum -y install kubelet kubeadm kubectl --disableexcludes=kubernetes
cat <<EOF >> /etc/default/kubelet
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --container-runtime=remote --container-runtime-endpoint="unix:///var/run/crio/crio.sock"
EOF
systemctl enable --now kubelet
```
# Configure firewall
- Ref: https://kubernetes.io/docs/reference/ports-and-protocols/
- Ref: https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md#vxlan
```console
firewall-cmd --add-port 2379-2380/tcp --permanent
firewall-cmd --add-port 6443/tcp --permanent
firewall-cmd --add-port 8472/udp --permanent
firewall-cmd --add-port 10250/tcp --permanent
firewall-cmd --add-port 10257/tcp --permanent
firewall-cmd --add-port 10259/tcp --permanent
firewall-cmd --add-port 30000-32767/tcp --permanent
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
```
# Create Kubernetes Cluster
- The `pull` command downloads the required container images, this is optional as the `init` command will also download the images if they are not already present
- The `--pod-network-cidr` option for `init` command is required for Flannel networking
- Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
```console
kubeadm config images pull
kubeadm init --pod-network-cidr 10.244.0.0/16
```
# Configure kubectl admin login and allow pods to run on master (single-node Kubernetes)
```console
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/master-
```
# Install Flannel networking
```console
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
- In case of `"cni0" already has an IP address different from 10.244.0.1/24` error
- This error occurs when you deploy a pod immediately after creating the Kubernetes cluster
- You can either reboot the node, or run below commands to recreate the CNI
```console
ip link del cni0
ip link del flannel.1
kubectl delete pod --selector=app=flannel -n kube-system
kubectl delete pod --selector=k8s-app=kube-dns -n kube-system
```
