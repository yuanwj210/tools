new k8s
1.准备工作（安装依赖，设置系统配置等）
nginx.conf modules-k8s.conf crictl.yaml k8s.conf 99-kubernetes-cri.conf
cat >nginx.conf
load_module /usr/lib/nginx/modules/ngx_stream_module.so;
worker_processes 4;
events {
        multi_accept on;
        use epoll;
        worker_connections 10240;
}

stream {
    upstream kube_apiserver {
        least_conn;
        server 10.136.213.9:5443;
    }

    server {
        listen        0.0.0.0:6443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 3s;
    }
}

cat >modules-k8s.conf
br_netfilter
overlay
ip_vs
ip_vs_rr  
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4

cat >crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false

cat >k8s.conf 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

cat >99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1


cat >main.yaml
---
- hosts: all
  vars:
    - {"zabbix_server" : "113.142.1.188,209.177.83.232,113.142.1.191"}
  tasks:

  - name: "disable swapoff"
    shell: "swapoff -a"

  - name: "delete fstab swap line"
    shell: sed -i '/swap.img/d' /etc/fstab

  - name: "import apt gpg"
    shell: "sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg"

  - name: "import apt sourcelist"
    shell: echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list
.d/kubernetes.list


  - name: "install kube* containd nginx"
    apt:
     name: "{{ item }}"
     update_cache: true
    loop: 
     - "nginx"
     - "containerd"
     - "kubelet=1.19.4-00"
     - "kubectl=1.19.4-00"
     - "kubeadm=1.19.4-00"

  - name: "apt mark"
    shell: "sudo apt-mark hold kubelet kubeadm kubectl"

  - name: "cp nginx conf"
    copy:
     src: "./nginx.conf"
     dest: "/etc/nginx"

  - name: "restart nginx"
    shell: "nginx -t && nginx -s reload"
    

  - name: "cp modules conf"
    copy:
     src: "./modules-k8s.conf"
     dest: "/etc/modules-load.d/k8s.conf"

  - name: "cp crictl conf"
    copy:
     src: "./crictl.yaml"
     dest: "/etc/"

  - name: "cp sysctl conf"
    copy:
     src: "./k8s.conf"
     dest: "/etc/sysctl.d/k8s.conf"

  - name: "cp 99-kubernetes-cri.conf"
    copy:
     src: "./99-kubernetes-cri.conf"
     dest: "/etc/sysctl.d/"

  - name: "sysctl"
    shell: "sudo sysctl --system"

  - name: "modprobe "
    shell: modprobe br_netfilter
  
  - name: "containerd config"
    shell: mkdir -p /etc/containerd && containerd config default  >/etc/containerd/config.toml
    ignore_errors: true

  - name: "replace containerd cgroup"
    shell: sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

  - name: "restart containerd "
    shell: systemctl restart containerd 

cat >update.yaml 
---
- hosts: all
  vars:
    - {"zabbix_server" : "113.142.1.188,209.177.83.232,113.142.1.191"}
  tasks:


  - name: "disable swapoff"
    shell: "sudo apt remove -y kubeadm=1.23.0-00 kubectl=1.23.0-00 kubelet=1.23.0-00"


  - name: "install kube* containd nginx"
    apt:
     name: "{{ item }}"
     update_cache: true
    loop: 
     - "kubelet=1.19.4-00"
     - "kubectl=1.19.4-00"
     - "kubeadm=1.19.4-00"

#三主三从，需单独部署etcd集群（用于初始化k8的yaml文件）
cat >kubeadm.yml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.31.0.222
  bindPort: 5443
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
  certSANs:
    - 127.0.0.1
    - 172.31.4.73
    - 172.31.8.130
    - 172.31.0.222
    - 54.219.205.205
    - 54.183.214.3
    - 52.53.177.56
  extraArgs:
    feature-gates: TTLAfterFinished=true
    service-node-port-range: "20000-32379"
  extraVolumes:
    - name: localtime
      hostPath: /etc/localtime
      mountPath: /etc/localtime
      readOnly: true
      pathType: File
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager:
  extraArgs:
    feature-gates: TTLAfterFinished=true
  extraVolumes:
    - hostPath: /etc/localtime
      mountPath: /etc/localtime
      name: localtime
      readOnly: true
      pathType: File
etcd:
  external:
    endpoints:
    - "https://172.31.4.73:2379"
    - "https://172.31.8.130:2379"
    - "https://172.31.0.222:2379"
    caFile: "/etc/etcd/ssl/ca.pem"
    certFile: "/etc/etcd/ssl/etcd.pem"
    keyFile: "/etc/etcd/ssl/etcd-key.pem"

imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.19.4
controlPlaneEndpoint: 172.31.0.222:6443
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.97.0.0/12
  podSubnet: 10.244.0.0/12
scheduler:
  extraArgs:
    feature-gates: TTLAfterFinished=true
  extraVolumes:
    - hostPath: /etc/localtime
      mountPath: /etc/localtime
      name: localtime
      readOnly: true
      pathType: File
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
bindAddressHardFail: false
clientConnection:
  acceptContentTypes: ""
  burst: 0
  contentType: ""
  kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
  qps: 0
clusterCIDR: ""
configSyncPeriod: 0s
conntrack:
  maxPerCore: null
  min: null
  tcpCloseWaitTimeout: null
  tcpEstablishedTimeout: null
detectLocalMode: ""
enableProfiling: false
healthzBindAddress: ""
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: null
  minSyncPeriod: 0s
  syncPeriod: 0s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 5s
  scheduler: "wrr"
  strictARP: false
  syncPeriod: 5s
  tcpFinTimeout: 0s
  tcpTimeout: 0s
  udpTimeout: 0s
kind: KubeProxyConfiguration
metricsBindAddress: ""
mode: "ipvs"
---
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
cgroupDriver: systemd
logging: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
---

#一主两从，自动安装etcd
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.136.213.9
  bindPort: 5443
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
  certSANs:
    - 127.0.0.1
    - 10.136.213.9
    - 165.22.186.229
    - 138.197.16.23
    - 138.197.31.24
  extraArgs:
    feature-gates: TTLAfterFinished=true
    service-node-port-range: "20000-32379"
  extraVolumes:
    - name: localtime
      hostPath: /etc/localtime
      mountPath: /etc/localtime
      readOnly: true
      pathType: File
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager:
  extraArgs:
    feature-gates: TTLAfterFinished=true
  extraVolumes:
    - hostPath: /etc/localtime
      mountPath: /etc/localtime
      name: localtime
      readOnly: true
      pathType: File

imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: v1.19.4
controlPlaneEndpoint: 10.136.213.9:6443
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.97.0.0/12
  podSubnet: 10.244.0.0/12
scheduler:
  extraArgs:
    feature-gates: TTLAfterFinished=true
  extraVolumes:
    - hostPath: /etc/localtime
      mountPath: /etc/localtime
      name: localtime
      readOnly: true
      pathType: File
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
bindAddressHardFail: false
clientConnection:
  acceptContentTypes: ""
  burst: 0
  contentType: ""
  kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
  qps: 0
clusterCIDR: ""
configSyncPeriod: 0s
conntrack:
  maxPerCore: null
  min: null
  tcpCloseWaitTimeout: null
  tcpEstablishedTimeout: null
detectLocalMode: ""
enableProfiling: false
healthzBindAddress: ""
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: null
  minSyncPeriod: 0s
  syncPeriod: 0s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 5s
  scheduler: "wrr"
  strictARP: false
  syncPeriod: 5s
  tcpFinTimeout: 0s
  tcpTimeout: 0s
  udpTimeout: 0s
kind: KubeProxyConfiguration
metricsBindAddress: ""
mode: "ipvs"
---
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
cgroupDriver: systemd
logging: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
---


1.mester节点操作
#stop systemd-resolved
systemctl stop systemd-resolved.service
systemctl disable systemd-resolved.service
echo nameserver 8.8.8.8 > /etc/resolv.conf

#初始化k8
kubeadm init --config ./kubeadm.yml   --upload-certs  --skip-phases=addon/kube-proxy

#安装helm并使用helm安装cilium
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
bash ./get_helm.sh 
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.11.0   --namespace kube-system   --set kubeProxyReplacement=strict   --set k8sServiceHost=10.136.213.9   --set k8sServicePort=6443

#install ingress 
tar -xf ingress-nginx-4.1.4.tgz 
cd ingress-nginx/
vim values.yaml
kubectl create ns ingress-nginx
helm install ingress-nginx  ingress-nginx -f ./ingress-nginx/values.yaml -n ingress-nginx
 
#查看contained运行的容器
crictl ps
#重置k8s集群
kubeadm reset


2.master添加副本节点
systemctl stop systemd-resolved.service
systemctl disable systemd-resolved.service
echo nameserver 8.8.8.8 > /etc/resolv.conf
kubeadm join 10.136.213.9:6443 --token abcdef.0123456789abcdef     --discovery-token-ca-cert-hash
sha256:cbf6212260fe7c5b0c5483a3fc2b33451d3dc535c53a919c548a01ead14315ce

