# kubernetes-the-kubeadm-way
Set up a Kubernetes cluster using kubeadm.

This is based on https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/.


## Prerequisite

GCP account is required. It is recommended to create a dedicated project.


## Single control-plane cluster

### 1. Create instances

Open https://console.cloud.google.com/compute and create the following instances:

- `master1`
- `worker1`

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
  --image-project=debian-cloud --image-family=debian-10 \
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
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
kubectl version --client
kubeadm version
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


### 4. Join the cluster (worker)

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


### 6. Reconfigure the cluster (master)

Here set up OpenID Connect authentication.

```yaml
# cluster.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
apiServer:
  extraArgs:
    oidc-issuer-url: https://accounts.google.com
    oidc-client-id: REDUCTED.apps.googleusercontent.com
```

```sh
sudo kubeadm upgrade apply --config=cluster.yaml
```

Make sure you can access the cluster using kubelogin.

```yaml
# .kube/config
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --grant-type=authcode-keyboard
      - --oidc-issuer-url=https://accounts.google.com
      - --oidc-client-id=REDUCTED.apps.googleusercontent.com
      - --oidc-client-secret=REDUCTED
```

```console
$ kubectl --user=oidc get nodes
Error from server (Forbidden): nodes is forbidden: User "https://accounts.google.com#REDUCTED" cannot list resource "nodes" in API group "" at the cluster scope
```


### 7. Destroy the cluster (master)

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


## Highly available cluster

### 1. Create instances

Open https://console.cloud.google.com/compute and create the following instances:

- `master1`
- `master2`
- `master3`
- `worker1`

with the following options:

- Your nearest region and zone
- e2-standard-2 (2 vCPUs, 8 GB memory)
- Debian 10
- Default network
- Network tags: `api` (master) or `worker` (worker)

As well as you can create instances on CLI.

```sh
gcloud beta compute instances create master1 --zone=asia-northeast1-b \
  --machine-type=e2-standard-2 \
  --subnet=default \
  --tags=api \
  --no-service-account --no-scopes \
  --image-project=debian-cloud --image-family=debian-10 \
  --boot-disk-size=10GB --boot-disk-type=pd-standard

gcloud beta compute instances create master2 --zone=asia-northeast1-b \
  --machine-type=e2-standard-2 \
  --subnet=default \
  --tags=api \
  --no-service-account --no-scopes \
  --image-project=debian-cloud --image-family=debian-10 \
  --boot-disk-size=10GB --boot-disk-type=pd-standard

gcloud beta compute instances create master3 --zone=asia-northeast1-b \
  --machine-type=e2-standard-2 \
  --subnet=default \
  --tags=api \
  --no-service-account --no-scopes \
  --image-project=debian-cloud --image-family=debian-10 \
  --boot-disk-size=10GB --boot-disk-type=pd-standard

gcloud beta compute instances create worker1 --zone=asia-northeast1-b \
  --machine-type=e2-standard-2 \
  --subnet=default \
  --tags=worker \
  --no-service-account --no-scopes \
  --image-project=debian-cloud --image-family=debian-10 \
  --boot-disk-size=10GB --boot-disk-type=pd-standard
```

You can log in to the instances on the console.


### 2. Create a load balancer

Open https://console.cloud.google.com/net-services/loadbalancing and create a load balancer.

- Frontend
  - A ephemeral IP address
  - Port 6443
- Backend
  - Add `master1` (DO NOT add `master2` and `master3` at this time)
  - No health check

Open https://console.cloud.google.com/networking/firewalls and create a rule.

- Direction: Ingress
- Target tags: `api`
- Source IP ranges: `0.0.0.0/0`
- Protocols and ports: tcp/6443


### 3. Install the tools (master and worker)

See the above section.


### 4. Create a cluster (master1)

Create a cluster on master1.

```sh
sudo kubeadm init --control-plane-endpoint "35.213.105.207:6443" --upload-certs
```

You can see the join commands.

```
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 35.213.105.207:6443 --token eivrbe.64tv35ww4apkdvdj \
    --discovery-token-ca-cert-hash sha256:be4c5568226583d231cea663a7d6c68df8b509e7f06fc3427ec531462ee98ef0 \
    --control-plane --certificate-key 6487107f16a7d466111e317010e8082d48b26c0e7e383b5fb14c23ec05ec2531

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 35.213.105.207:6443 --token eivrbe.64tv35ww4apkdvdj \
    --discovery-token-ca-cert-hash sha256:be4c5568226583d231cea663a7d6c68df8b509e7f06fc3427ec531462ee98ef0 
```

Deploy the CNI network plugin.

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml

kubectl -n kube-system get cm kubeadm-config -oyaml
```

You can see the cluster config.

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
    controlPlaneEndpoint: 35.213.105.207:6443
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
        advertiseAddress: 10.146.0.8
        bindPort: 6443
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: ClusterStatus
kind: ConfigMap
```

Make sure master1 is ready.

```console
$ kubectl get nodes
NAME      STATUS   ROLES    AGE    VERSION
master1   Ready    master   4m2s   v1.18.3
```


### 5. Join the cluster (master2/3)

Run the join command on master2 and master3.

```sh
sudo kubeadm join 35.213.105.207:6443 --token eivrbe.64tv35ww4apkdvdj \
    --discovery-token-ca-cert-hash sha256:be4c5568226583d231cea663a7d6c68df8b509e7f06fc3427ec531462ee98ef0 \
    --control-plane --certificate-key 6487107f16a7d466111e317010e8082d48b26c0e7e383b5fb14c23ec05ec2531
```

Finally make sure all masters are ready.

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

Open https://console.cloud.google.com/net-services/loadbalancing and add `master2` and `master3` to the backend.


### 6. Join the cluster (worker1)

See the above section.

Make sure all nodes are ready.

```console
% kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   62m     v1.18.3
master2   Ready    master   12m     v1.18.3
master3   Ready    master   14m     v1.18.3
worker1   Ready    <none>   3m26s   v1.18.3
```



