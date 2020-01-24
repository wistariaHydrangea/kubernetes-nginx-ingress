# L7 Loadblancer construction on kubernetes

Let's build a kubernetes environment with a VMware ESXi server in the home on-premise environment.
Many of the articles that are falling are built under the environment of public cloud (GCP, AWS, etc.), and the construction method and manifest writing method may be slightly different between on-premise and cloud.

## operating environment

- VMware ESXi 6.7.0 Update2
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

```terminal
$ sudo setenforce 0
$ sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## 2. SWAP Invalidation

```terminal
$ sudo swapoff -a
$ sudo vim /etc/fstab
/dev/mapper/centos-swap swap         swap    defaults        0 0
                            ↓
#/dev/mapper/centos-swap swap         swap    defaults        0 0
```

## 3. Firewalld Invalidation

```terminal
$ sudo systemctl stop firewalld
$ sudo systemctl disable firewalld
```

## 4. Fix IP to NIC

```txt
```

## 5. OS pachage update

```terminal
$ sudo yum -y update
```

## Docker install & start-up

Install and launch the latest version of Docker from the official repository.

```terminal
$ sudo yum -y install yum-utils device-mapper-persistent-data lvm2
$ sudo yum-config-manager \
    --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum -y install docker-ce
$ sudo systemctl enable docker && sudo systemctl start docker
```

## Install of kubeadm & kubelet & kubectl

Set kubernetes repository.

```terminal
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

```terminal
$ sudo sh -c "cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
"

$ sudo sysctl --system
```

## master node set

### Initialization of master node

Initialize master with __kube in command.
The meaning of the parameters is as follows.

--apiserver-advertise-address: Specify the Listen IP address of APIServer, usually the IP address of eth0, which is the NIC of Master node
--pod-network-cidr: CIDR of the internal IP address to be assigned to Pod, or 10.244.0.0/16 if Flannel described later is used for CNI
--service-cidr: CIDR of internal IP address assigned to Service, default is 10.96.0.0/12

```terminal
$ sudo kubeadm init \
    --apiserver-advertise-address 172.16.10.160 \
    --pod-network-cidr 10.244.0.0/16 \
    --service-cidr 172.16.130.0/24
```

If you want to cancel after joining the cluster with __kubeadm join__, use __kubeadm reset__.
Note that after doing __kubeadm reset__, you need to remove the CNI added during __kubeadm join__.

```terminal
$ kubeadm reset
$ ip link delete cni0
$ ip link delete flannel.1
```

Execute the following command to make kubectl work for non-root user.

```terminal
$ sudo mkdir -p $HOME/.kube
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
kube-system      etcd-master-node.example.com                           1/1     Running   0          7d
kube-system      kube-apiserver-master-node.example.com                 1/1     Running   0          7d
kube-system      kube-controller-manager-master-node.example.com        1/1     Running   0          7d
kube-system      kube-flannel-ds-amd64-c6gwk                             1/1     Running   0          7d
kube-system      kube-flannel-ds-amd64-rxc6b                             1/1     Running   0          7d
kube-system      kube-flannel-ds-amd64-t9hjw                             1/1     Running   0          7d
kube-system      kube-flannel-ds-amd64-v6hcd                             1/1     Running   0          7d
```

Install metric server

```terminal
$ git clone https://github.com/kubernetes-sigs/metrics-server.git
```

Even if __kubectl apply__ as it is, it can not be started normally, so add `command` part.
`Node-role.kubernetes.io/master: ""` is required to use the __kubectl top__ command.

(metrics-server-deployment.yaml)

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.6
        args:
          - --cert-dir=/tmp
          - --secure-port=4443
        ports:
        - name: main-port
          containerPort: 4443
          protocol: TCP
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        imagePullPolicy: Always
        ### 追加開始
        command:
        - /metrics-server
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalDNS,InternalIP,ExternalDNS,ExternalIP,Hostname
        ### 追加終了
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      nodeSelector:
        ### 追加開始
        node-role.kubernetes.io/master: ""
        ### 追加終了
        beta.kubernetes.io/os: linux
```

Deploy metric server

```terminal
$ kubectl apply -f metric-server/deploy/+1.8/
```

## Deploy of metalLB

Deploy metallb

```terminal
$ wget https://raw.githubusercontent.com/danderson/metallb/master/manifests/metallb.yaml
$ kubectl apply -f metallb.yaml
```

MetalLB has two operation modes, BGP mode and L2 mode.
In BGP mode, a BGP-compatible network device is required upstream of the cluster network, so this time prepare a manifest metallb-l2.yaml that is configured to operate MetalLB in L2 mode.
For addresses in the manifest, specify the External IP address range to be assigned to Service.
Note that the IP address range is specified by the IP address range obtained by the physical server.

(metallb-l2.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: my-ip-space
      protocol: layer2
      addresses:
      - 172.16.10.160-172.16.10.180
```

## Install of nginx Ingress controller

```terminal
$ wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
$ wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud-generic.yaml
$ kubectl apply -f mandatory.yaml
$ kubectl apply -f cloud-generic.yaml
```

Check if an external IP address is assigned to EXTERNAL-IP

```terminal
$ kubectl get all,node,ingress,configmap -A -o wide
NAMESPACE        NAME                                                       READY   STATUS    RESTARTS   AGE     IP              NODE                            NOMINATED NODE   READINESS GATES
ingress-nginx    pod/nginx-ingress-controller-568867bf56-jb65q              1/1     Running   0          16m     10.244.2.4      worker-node2.example.com   <none>           <none>
kube-system      pod/coredns-5644d7b6d9-jqxfd                               1/1     Running   0          147m    10.244.0.3      master-node.example.com    <none>           <none>
kube-system      pod/coredns-5644d7b6d9-px7tg                               1/1     Running   0          147m    10.244.0.2      master-node.example.com    <none>           <none>
kube-system      pod/etcd-master-node.example.com                           1/1     Running   0          146m    172.16.10.114   master-node.example.com    <none>           <none>
kube-system      pod/kube-apiserver-master-node.example.com                 1/1     Running   0          146m    172.16.10.114   master-node.example.com    <none>           <none>
kube-system      pod/kube-controller-manager-master-node.example.com        1/1     Running   0          146m    172.16.10.114   master-node.example.com    <none>           <none>
kube-system      pod/kube-flannel-ds-amd64-7cggf                            1/1     Running   0          138m    172.16.10.116   worker-node1.example.com   <none>           <none>
kube-system      pod/kube-flannel-ds-amd64-l6vww                            1/1     Running   0          134m    172.16.10.117   worker-node2.example.com   <none>           <none>
kube-system      pod/kube-flannel-ds-amd64-rnt5x                            1/1     Running   0          142m    172.16.10.114   master-node.example.com    <none>           <none>
kube-system      pod/kube-proxy-4swl5                                       1/1     Running   0          147m    172.16.10.114   master-node.example.com    <none>           <none>
kube-system      pod/kube-proxy-dmgdj                                       1/1     Running   0          134m    172.16.10.117   worker-node2.example.com   <none>           <none>
kube-system      pod/kube-proxy-jz2cz                                       1/1     Running   0          138m    172.16.10.116   worker-node1.example.com   <none>           <none>
kube-system      pod/kube-scheduler-master-node.example.com                 1/1     Running   0          146m    172.16.10.114   master-node.example.com    <none>           <none>
metallb-system   pod/controller-57967b9448-khxpd                            1/1     Running   0          19m     10.244.1.6      worker-node1.example.com   <none>           <none>
metallb-system   pod/speaker-4nhtk                                          1/1     Running   0          19m     172.16.10.116   worker-node1.example.com   <none>           <none>
metallb-system   pod/speaker-nx7cd                                          1/1     Running   0          19m     172.16.10.114   master-node.example.com    <none>           <none>
metallb-system   pod/speaker-wc5gg                                          1/1     Running   0          19m     172.16.10.117   worker-node2.example.com   <none>           <none>
ns-test          pod/httpd-75cb4864cc-trljc                                 1/1     Running   0          2m30s   10.244.1.9      worker-node1.example.com   <none>           <none>
ns-test          pod/nginx-5c559d5697-mtgj5                                 1/1     Running   0          2m30s   10.244.1.10     worker-node1.example.com   <none>           <none>

NAMESPACE       NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE     SELECTOR
default         service/kubernetes      ClusterIP      10.96.0.1        <none>          443/TCP                      147m    <none>
ingress-nginx   service/ingress-nginx   LoadBalancer   10.108.101.67    172.16.10.160   80:30325/TCP,443:31014/TCP   16m     app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/part-of=ingress-nginx
kube-system     service/kube-dns        ClusterIP      10.96.0.10       <none>          53/UDP,53/TCP,9153/TCP       147m    k8s-app=kube-dns
ns-test         service/apache-svc      NodePort       10.97.5.55       <none>          80:32296/TCP                 2m30s   app=httpd
ns-test         service/nginx-svc       NodePort       10.103.154.201   <none>          80:31528/TCP                 2m30s   app=nginx

NAMESPACE        NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE    CONTAINERS     IMAGES                                   SELECTOR
kube-system      daemonset.apps/kube-flannel-ds-amd64     3         3         3       3            3           <none>                        142m   kube-flannel   quay.io/coreos/flannel:v0.11.0-amd64     app=flannel
kube-system      daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           <none>                        142m   kube-flannel   quay.io/coreos/flannel:v0.11.0-arm       app=flannel
kube-system      daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           <none>                        142m   kube-flannel   quay.io/coreos/flannel:v0.11.0-arm64     app=flannel
kube-system      daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           <none>                        142m   kube-flannel   quay.io/coreos/flannel:v0.11.0-ppc64le   app=flannel
kube-system      daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           <none>                        142m   kube-flannel   quay.io/coreos/flannel:v0.11.0-s390x     app=flannel
kube-system      daemonset.apps/kube-proxy                3         3         3       3            3           beta.kubernetes.io/os=linux   147m   kube-proxy     k8s.gcr.io/kube-proxy:v1.16.2            k8s-app=kube-proxy
metallb-system   daemonset.apps/speaker                   3         3         3       3            3           beta.kubernetes.io/os=linux   19m    speaker        metallb/speaker:master                   app=metallb,component=speaker

NAMESPACE        NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS                 IMAGES                                                                  SELECTOR
ingress-nginx    deployment.apps/nginx-ingress-controller   1/1     1            1           16m     nginx-ingress-controller   quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1   app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/part-of=ingress-nginx
kube-system      deployment.apps/coredns                    2/2     2            2           147m    coredns                    k8s.gcr.io/coredns:1.6.2                                                k8s-app=kube-dns
metallb-system   deployment.apps/controller                 1/1     1            1           19m     controller                 metallb/controller:master                                               app=metallb,component=controller
ns-test          deployment.apps/httpd                      1/1     1            1           2m30s   httpd                      httpd:alpine                                                            app=httpd
ns-test          deployment.apps/nginx                      1/1     1            1           2m30s   nginx                      nginx:alpine                                                            app=nginx

NAMESPACE        NAME                                                  DESIRED   CURRENT   READY   AGE     CONTAINERS                 IMAGES                                                                  SELECTOR
ingress-nginx    replicaset.apps/nginx-ingress-controller-568867bf56   1         1         1       16m     nginx-ingress-controller   quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1   app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/part-of=ingress-nginx,pod-template-hash=568867bf56
kube-system      replicaset.apps/coredns-5644d7b6d9                    2         2         2       147m    coredns                    k8s.gcr.io/coredns:1.6.2                                                k8s-app=kube-dns,pod-template-hash=5644d7b6d9
metallb-system   replicaset.apps/controller-57967b9448                 1         1         1       19m     controller                 metallb/controller:master                                               app=metallb,component=controller,pod-template-hash=57967b9448
ns-test          replicaset.apps/httpd-75cb4864cc                      1         1         1       2m30s   httpd                      httpd:alpine                                                            app=httpd,pod-template-hash=75cb4864cc
ns-test          replicaset.apps/nginx-5c559d5697                      1         1         1       2m30s   nginx                      nginx:alpine                                                            app=nginx,pod-template-hash=5c559d5697

NAMESPACE   NAME                                 STATUS   ROLES    AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
            node/master-node.example.com    Ready    master   147m   v1.16.2   172.16.10.114   <none>        CentOS Linux 7 (Core)   3.10.0-1062.1.2.el7.x86_64   docker://19.3.4
            node/worker-node1.example.com   Ready    <none>   138m   v1.16.2   172.16.10.116   <none>        CentOS Linux 7 (Core)   3.10.0-1062.1.2.el7.x86_64   docker://19.3.4
            node/worker-node2.example.com   Ready    <none>   134m   v1.16.2   172.16.10.117   <none>        CentOS Linux 7 (Core)   3.10.0-1062.1.2.el7.x86_64   docker://19.3.4

NAMESPACE   NAME                    HOSTS                    ADDRESS         PORTS   AGE
ns-test     ingress.extensions/lb   nginx.example.com   172.16.10.160   80      2m30s
```

Create Ingress and app manifest and check operation

(testApp-ingress.yaml)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ns-test

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: lb
  namespace: ns-test
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
    - host: nginx.example.com
      http:
        paths:
          - path: /apache
            backend:
              serviceName: apache-svc
              servicePort: 80
          - path: /nginx
            backend:
              serviceName: nginx-svc
              servicePort: 80
          - path: /
            backend:
              serviceName: blackhole
              servicePort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: apache-svc
  namespace: ns-test
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: httpd
  type: NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
  namespace: ns-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - image: httpd:alpine
        name: httpd
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: ns-test
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: ns-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
        ports:
        - containerPort: 80

```

Editing now...

## Reference material

1. [Kubernetes on CentOS7 - Qiita  (https://qiita.com/nagase/items/15726e37057e7cc3b8cd)](https://qiita.com/nagase/items/15726e37057e7cc3b8cd)
2. [kubeadmのインストール - Kubernetes  (https://kubernetes.io/ja/docs/setup/independent/install-kubeadm/)](https://kubernetes.io/ja/docs/setup/independent/install-kubeadm/)
3. [kubeadmを使用したシングルマスタークラスターの作成 - Kubernetes  (https://kubernetes.io/ja/docs/setup/independent/create-cluster-kubeadm/)](https://kubernetes.io/ja/docs/setup/independent/create-cluster-kubeadm/)
4. [2019年版・Kubernetesクラスタ構築入門 | さくらのナレッジ  (https://knowledge.sakura.ad.jp/20955/)](https://knowledge.sakura.ad.jp/20955/)
5. [kubernetesにNGINX Ingress Controllerを追加して、ホスト名で ...  (https://qiita.com/megasys1968/items/aa94ddfb92f57e08c4dd)](https://qiita.com/megasys1968/items/aa94ddfb92f57e08c4dd)
6. [MetalLB, bare metal load-balancer for Kubernetes  (https://metallb.universe.tf/)](https://metallb.universe.tf/)
7. [ingress-nginx/index.md at master · kubernetes/ingress-nginx ...  (https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md)](https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md)
8. [Installation Guide - NGINX Ingress Controller - Kubernetes  (https://kubernetes.github.io/ingress-nginx/deploy/)](https://kubernetes.github.io/ingress-nginx/deploy/)
9. [オンプレミスのKubernetesにNginx-Ingress-Controllerを設定する ...  (https://qiita.com/soumi/items/c6358e5e859004c2961c)](https://qiita.com/soumi/items/c6358e5e859004c2961c)
10. [Ingress - Kubernetes  (https://kubernetes.io/docs/concepts/services-networking/ingress/)](https://kubernetes.io/docs/concepts/services-networking/ingress/)
11. [kubernetesにNginx Ingress Controllerをセットアップ - Qiita  (https://qiita.com/sheepland/items/37ece5800eb6972ffd91)](https://qiita.com/sheepland/items/37ece5800eb6972ffd91)
12. [オンプレでもIngressする - Qiita  (https://qiita.com/murata-tomohide/items/801c25492e55f672c23c)](https://qiita.com/murata-tomohide/items/801c25492e55f672c23c)
13. [Kubernetes に NGINX Ingress Controller をデプロイする方法 ...  (https://garafu.blogspot.com/2019/06/install-nginxingress.html)](https://garafu.blogspot.com/2019/06/install-nginxingress.html)
14. [kubernetes/ingress-nginx: NGINX Ingress Controller ... - GitHub  (https://github.com/kubernetes/ingress-nginx)](https://github.com/kubernetes/ingress-nginx)
15. [DockerとKubernetesのPodのネットワーキングについてまとめました  (https://foobaron.hatenablog.com/entry/k8s-pod-networking)](https://foobaron.hatenablog.com/entry/k8s-pod-networking)
16. [Kubernetes: Flannel networking - Han's blog  (https://blog.laputa.io/kubernetes-flannel-networking-6a1cb1f8ec7c)](https://blog.laputa.io/kubernetes-flannel-networking-6a1cb1f8ec7c)
17. [オンプレミスKubernetes のネットワーク - JANOG  (https://www.janog.gr.jp/meeting/janog43/application/files/2215/4900/5049/janog43-k8s-shirota.pdf)](https://www.janog.gr.jp/meeting/janog43/application/files/2215/4900/5049/janog43-k8s-shirota.pdf)
18. [Debugging Kubernetes Networking | Eficode Praqma  (https://www.praqma.com/stories/debugging-kubernetes-networking/)](https://www.praqma.com/stories/debugging-kubernetes-networking/)
19. [オンプレでもIngressする - Qiita  (https://qiita.com/murata-tomohide/items/801c25492e55f672c23c)](https://qiita.com/murata-tomohide/items/801c25492e55f672c23c)
20. [Kubernetesクラスタでmetrics-serverを導入してkubectl topや ...  (https://qiita.com/chataro0/items/28f8744e2781f730a0e6)](https://qiita.com/chataro0/items/28f8744e2781f730a0e6)
21. [オンプレでもIngressする - Qiita  (https://qiita.com/murata-tomohide/items/801c25492e55f672c23c)](https://qiita.com/murata-tomohide/items/801c25492e55f672c23c)
22. [Kubernetesクラスタでmetrics-serverを導入してkubectl topや ...  (https://qiita.com/chataro0/items/28f8744e2781f730a0e6)](https://qiita.com/chataro0/items/28f8744e2781f730a0e6)
23. [TLS/HTTPS - NGINX Ingress Controller - Kubernetes  (https://kubernetes.github.io/ingress-nginx/user-guide/tls/)](https://kubernetes.github.io/ingress-nginx/user-guide/tls/)
