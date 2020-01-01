# L7 Loadblancer construction on kubernetes

Let's build a kubernetes environment with a VMware ESXi server in the home on-premise environment.
Many of the articles that are falling are built under the environment of public cloud (GCP, AWS, etc.), and the construction method and manifest writing method may be slightly different between on-premise and cloud.

## operating environment

- VMware ESXi 6.7.0 Update2
  - router node
    - CPU 2Core
    - RAM 2GB(Minimum requirements 2GB)
  - master node
    - CPU 2Core
    - RAM 2GB(Minimum requirements 2GB)
  - worker node1
    - CPU 2Core
    - RAM 2GB(Minimum requirements 2GB)
  - worker node2
    - CPU 2Core
    - RAM 2GB(Minimum requirements 2GB)

## Preparing the VM

Perform the following tasks on all VMs

1. SELinux Invalidation
2. SWAP Invalidation
3. Firewalld Invalidation ( If not disabled, open ports as needed )
4. Fix IP to NIC
5. OS pachage update

## 1. SELinux Invalidation

The official document of kubernetes says `You have to do this until SELinux support is improved in the kubelet.`, and SELinux needs to be disabled because kubelet is not supported.

```txt
# setenforce 0
# sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## 2. SWAP Invalidation

```txt
$ sudo swapoff -a
$ sudo vim /etc/fstab
/dev/mapper/centos-swap swap         swap    defaults        0 0
                            ↓
#/dev/mapper/centos-swap swap         swap    defaults        0 0
```

## 3. Firewalld Invalidation

```txt
$ sudo systemctl stop firewalld
$ sudo systemctl disable firewalld
```

4. Fix IP to NIC

```txt
```

5. OS pachage update

```txt
$ sudo yum -y update
```

## Docker install & start-up

Install and launch the latest version of Docker from the official repository.

```txt
$ sudo yum -y install yum-utils device-mapper-persistent-data lvm2
$ sudo yum-config-manager \
    --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum -y install docker-ce
$ sudo systemctl enable docker && sudo systemctl start docker
```

## Install of kubeadm & kubelet & kubectl

Set kubernetes repository.

```erpo
$ sudo sh -c "cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
"
```

Install kubeadm & kubelet & kubectl

```terminal
$ sudo yum -y install kubelet kubeadm kubectl
```

Enable and start kubelet automatically

```terminal
$ sudo systemctl enable kubelet && sudo systemctl start kubelet
```

RHEL / CentOS 7 users added a parameter to avoid bypassing iptables

```conf
$ sudo sh -c "cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
"

$ sudo sysctl --system
```

## master node set

### Initialization of master node

Initialize master with `kube init` command.
The meaning of the parameters is as follows.

--apiserver-advertise-address: Specify the Listen IP address of APIServer, usually the IP address of eth0, which is the NIC of Master node
--pod-network-cidr: CIDR of the internal IP address to be assigned to Pod, or 10.244.0.0/16 if Flannel described later is used for CNI
--service-cidr: CIDR of internal IP address assigned to Service, default is 10.96.0.0/12

```terminal
$ sudo kubeadm init \
    --apiserver-advertise-address 172.16.10.114 \
    --pod-network-cidr 10.244.0.0/16 \
    --service-cidr 172.16.130.0/24
```

Execute the following command to make kubectl work for non-root user.

```terminal
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install Fannel, one of the CNI plugins.

```terminal
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Check if all pods are running.

```terminal
$ kubectl get pod --all-namespaces
NAMESPACE        NAME                                                   READY   STATUS    RESTARTS   AGE
kube-system      coredns-5644d7b6d9-ws6lh                               1/1     Running   0          7d
kube-system      coredns-5644d7b6d9-x9vr8                               1/1     Running   0          7d
kube-system      etcd-master-node.kyukeispaces.com                      1/1     Running   0          7d
kube-system      kube-apiserver-master-node.kyukeispaces.com            1/1     Running   0          7d
kube-system      kube-controller-manager-master-node.kyukeispaces.com   1/1     Running   0          7d
kube-system      kube-flannel-ds-amd64-c6gwk                             1/1     Running   0          7d
kube-system      kube-flannel-ds-amd64-rxc6b                             1/1     Running   0          7d
kube-system      kube-flannel-ds-amd64-t9hjw                             1/1     Running   0          7d
kube-system      kube-flannel-ds-amd64-v6hcd                             1/1     Running   0          7d
```

Editing now...
