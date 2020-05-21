# kubernetes-the-kubeadm-way
Set up a Kubernetes cluster using kubeadm.

This is based on https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/.


## Prerequisite

GCP account is required. It is recommended to create a dedicated project.


## Single control-plane cluster

### 1. Create instances

Open https://console.cloud.google.com/compute and create the following instances:

- `master1`
- `node1`

with the following options:

- your nearest region and zone
- e2-standard-2 (2 vCPUs, 8 GB memory)
- default network
- Debian 10

As well as you can create instances on CLI.

```sh
gcloud beta compute instances create master1 --zone=asia-northeast1-b \
  --machine-type=e2-standard-2 \
  --subnet=default \
  --no-service-account --no-scopes \
  --image=debian-10-buster-v20200413 --image-project=debian-cloud \
  --boot-disk-size=10GB --boot-disk-type=pd-standard
```

You can log in to the instances on the console.


### 2. Install the tools (master and worker)

Install Docker.

```sh
sudo apt update
sudo apt install -y docker.io

sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo mkdir -p /etc/systemd/system/docker.service.d
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
sudo docker version
```

```console
Client:
 Version:           18.09.1
 API version:       1.39
 Go version:        go1.11.6
 Git commit:        4c52b90
 Built:             Tue, 03 Sep 2019 19:59:35 +0200
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          18.09.1
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.11.6
  Git commit:       4c52b90
  Built:            Tue Sep  3 17:59:35 2019
  OS/Arch:          linux/amd64
  Experimental:     false
```

Install kubeadm, kubelet and kubectl.

```sh
sudo apt install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

```console
$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.3", GitCommit:"2e7996e3e2712684bc73f0dec0200d64eec7fe40", GitTreeState:"clean", BuildDate:"2020-05-20T12:52:00Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}

$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.3", GitCommit:"2e7996e3e2712684bc73f0dec0200d64eec7fe40", GitTreeState:"clean", BuildDate:"2020-05-20T12:49:29Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
```

### 3. Create a cluster (master)

Create a cluster on master.

```sh
sudo kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

```console
NAME      STATUS     ROLES    AGE   VERSION
master1   NotReady   master   60s   v1.18.3
```

You can see the cluster config.

```sh
kubectl -n kube-system get cm kubeadm-config -oyaml
```

```yaml
apiVersion: v1
data:
  ClusterConfiguration: |
    apiServer:
      extraArgs:
        authorization-mode: Node,RBAC
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta2
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns:
      type: CoreDNS
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: k8s.gcr.io
    kind: ClusterConfiguration
    kubernetesVersion: v1.18.3
    networking:
      dnsDomain: cluster.local
      serviceSubnet: 10.96.0.0/12
    scheduler: {}
  ClusterStatus: |
    apiEndpoints:
      master1:
        advertiseAddress: 10.146.0.4
        bindPort: 6443
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: ClusterStatus
kind: ConfigMap
```

Deploy Calico to the cluster.

```sh
kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

Make sure that Calico pods are running and master is ready.

```console
$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-789f6df884-gd45t   1/1     Running   0          49s
kube-system   calico-node-2dfdc                          1/1     Running   0          49s
kube-system   coredns-66bff467f8-5wdvg                   1/1     Running   0          17m
kube-system   coredns-66bff467f8-m5bm6                   1/1     Running   0          17m
kube-system   etcd-master1                               1/1     Running   0          18m
kube-system   kube-apiserver-master1                     1/1     Running   0          18m
kube-system   kube-controller-manager-master1            1/1     Running   0          18m
kube-system   kube-proxy-zshbn                           1/1     Running   0          17m
kube-system   kube-scheduler-master1                     1/1     Running   0          18m

$ kubectl get nodes
NAME      STATUS   ROLES    AGE   VERSION
master1   Ready    master   18m   v1.18.3
```


### 4. Join (worker)

Add worker to the cluster.

```sh
sudo kubeadm join 10.146.0.2:6443 --token boi6zz.jd1b0wetv6n534d1 \
    --discovery-token-ca-cert-hash sha256:5a848700dc552dabfb9e26ac53e6bc8ba7b7a5f498e7b6eae1fca53a63055e7f 
```

You need to copy and paste the kubeconfig from master in order to access the cluster from worker.

```sh
mkdir .kube
vim .kube/config
```

Make sure worker is ready.

```console
$ kubectl get nodes
NAME      STATUS   ROLES    AGE    VERSION
master1   Ready    master   27m    v1.18.3
worker1   Ready    <none>   3m1s   v1.18.3
```


### 5. Deploy an application (worker)

Enable the bash completion.

```sh
source <(kubectl completion bash)
```

```sh
kubectl create deployment echoserver --image=gcr.io/google_containers/echoserver:1.10
kubectl expose deployment echoserver --type=NodePort --target-port=8080 --port=80
```

Deploy echoserver and make sure you can access it via a node port.

```console
$ kubectl describe svc echoserver
Name:                     echoserver
Namespace:                default
Labels:                   app=echoserver
Annotations:              <none>
Selector:                 app=echoserver
Type:                     NodePort
IP:                       10.104.19.168
Port:                     <unset>  80/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30797/TCP
Endpoints:                192.168.235.130:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

$ curl 10.146.0.3:30797

Hostname: echoserver-67d7995c9f-bg2jp

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.13.3 - lua: 10008

Request Information:
        client_address=10.146.0.3
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://10.146.0.3:8080/

Request Headers:
        accept=*/*
        host=10.146.0.3:30797
        user-agent=curl/7.58.0

Request Body:
        -no body in request-
```


### 6. Destroy the cluster (master)

Remove worker.

```console
$ kubectl delete deployment echoserver
$ kubectl drain worker1 --delete-local-data --force --ignore-daemonsets

$ kubectl get nodes
NAME      STATUS                     ROLES    AGE   VERSION
master1   Ready                      master   64m   v1.18.3
worker1   Ready,SchedulingDisabled   <none>   40m   v1.18.3

$ kubectl delete node worker1
```

Clean up the control plane.

```sh
sudo kubeadm reset
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```
