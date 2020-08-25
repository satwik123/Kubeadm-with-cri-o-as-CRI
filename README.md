
**Point to be noted:** *CRI-o & kubernetes version(major & minor) should be same*

***Instllation of CRI-O***

    yum -y update
    
    VERSION=1.18
    sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_7/devel:kubic:libcontainers:stable.repo
    sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_7/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo
    
    yum install cri-o
    
    rpm -qi cri-o
    systemctl enable --now cri-o

***Install crictl CLI***

    VERSION="v1.18.0"
    
    wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
    tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
    rm -f crictl-$VERSION-linux-amd64.tar.gz

> crictl info {   "status": {
>     "conditions": [
>       {
>         "type": "RuntimeReady",
>         "status": true,
>         "reason": "",
>         "message": ""
>       },
>       {
>         "type": "NetworkReady",
>         "status": true,
>         "reason": "",
>         "message": ""
>       }
>     ]   } }

***Configure CRI-O***

**Configure ipv4_forward**

    $ cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
    net.bridge.bridge-nf-call-iptables  = 1
    net.ipv4.ip_forward                 = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF
    $ sysctl --system
    $ modprobe br_netfilter
    $ echo '1' > /proc/sys/net/ipv4/ip_forward

Modify crio config /etc/crio/crio.conf by changing cgroup_manager from "systemd" to "cgroupfs" as well as add "docker.io" registry to pull unqualified images( or else pods with image containing no registry in name are stuck in ImageInspectError state)

#Cgroup management implementation used for the runtime.

    cgroup_manager = "cgroupfs"

#List of registries to be used when pulling an unqualified image (e.g.,
#"alpine:latest"). By default, registries is set to "docker.io" for
#compatibility reasons. Depending on your workload and usecase you may add more
 registries (e.g., "quay.io", "registry.fedoraproject.org",
"registry.opensuse.org", etc.).

    registries = [
      "quay.io",
      "docker.io"
    ]

Stop and start crio service

    $ systemctl stop crio
    $ systemctl start crio

***Install & configure kubeadm***

    $ setenforce 0
    $ sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
    $ reboot
    
    $ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF
    
    $ yum install kubeadm -y 
    
    $ systemctl enable kubelet
    $ systemctl start kubelet
    
    $ swapoff -a

Initialize kubeadm
------------------
Change **apiserver-advertise-address** as master node address

    kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=172.16.0.41 --cri-socket=unix:///var/run/crio/crio.sock --apiserver-bind-port=443






