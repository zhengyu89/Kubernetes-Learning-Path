# DAY 17: VM setup [Control plane & worker node]

## setup cluster with kubeadm, with /28 service CIDR

## Objective for today:

1. Setup control plane and worker node with kubelet, kubeadm, kubectl, container runtime, cillum ready. Setup k alias in bash.
2. Kubeadm join worker nodes to control panel.
3. Understand IPAM and CIDR allocation

---

## Install Kubelet Kubeadm and Kubectl

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

sudo apt-get install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

## Create k alias

```bash
cat <<'EOF' >> ~/.bashrc

source <(kubectl completion bash)

alias k=kubectl

complete -o default -F __start_kubectl k

EOF
```

## Setup Container runtime

```bash
sudo apt-get install -y containerd

kubeadm init
```

> `sudo kubeadm init --service-cidr=10.96.0.0/28 --pod-network-cidr=10.244.0.0/16` to pre-specify the cidr

```bash
sudo sysctl -w net.ipv4.ip_forward=1

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Install Cilium [Only for control-plane node]

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)

CLI_ARCH=amd64

if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi

curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum

sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin

rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install

// To Validate

cilium status --wait
```

## Kubeadm Join: Connect worker node to control-plane node

Worker node need:

1. CONTROL-PLANE-IP
2. BOOTSTRAP-TOKEN [temp auth to join the cluster]
3. discovery-token-ca-cert-hash [make sure the controlplane is real]

### To get it:

1. In controlplane node run `kubeadm token create --print-join-command`
2. Paste the command to worker node and execute
3. Validate by `k get nodes`

```bash
kubeadm join 10.90.4.161:6443 --token 04m860.sb9fpzktfxt25z6d --discovery-token-ca-cert-hash sha256:fcfd699a9a07bee924eb08ff9ad49c19a891643b4fe5fbf286e973f29f6d6b4a
```

> Port 6443 is used for Kubernetes API server.

---

# CIDR

CIDR is stand for Classless Inter-Domain Routing. You can think as IP + <allowed address range>

There is 3 type of CIDR widely use is Cluster Pod CIDR, Node Pod CIDR and Service CIDR.

Cillum will take ownership on managing Cluster Pod CIDR and Node Pod CIDR by using Cillium IPAM.

## Cillium IPAM CIDR allocation

### Cluster and Node pod CIDR

```bash
cilium install \
  --set ipam.mode=cluster-pool \
  --set ipam.operator.clusterPoolIPv4PodCIDRList=10.244.0.0/16 \
  --set ipam.operator.clusterPoolIPv4MaskSize=25
```

## Service CIDR extension

Service CIDR can extend,

`k get servicecidr` to get the default service cidr

In this lab, we already setup `--service-cidr=10.96.0.0/28` during kubeadm init. Hence the output for `kubectl get servicecidr` should be:

```text
NAME       CIDRS         AGE
kubernetes 10.96.0.0/28 17d
```

Total IP range = 16

Total available IP = 12

[10.96.0.1 by kubernetes, 10.96.0.10 by kube-dns, :0 and :15 is reserved for network/broadcast addr]

Let's demonstrate when we are not enough IP address, how we can extend the IP range by CIDR:

1. Create a newcidr1.yaml by vim newcidr1.yaml
2. Configure a ServiceCIDR Kind

```bash
apiVersion: networking.k8s.io/v1
kind: ServiceCIDR
metadata:
  name: newcidr1
spec:
  cidrs:
  - 10.96.0.0/24
```

3. kubectl get servicecidr to show the CIDR is correctly configured.
4. Validate by the code below:

```bash
for i in $(seq 13 16); do kubectl create service clusterip "test-$i" --tcp 80 -o json | jq -r .spec.clusterIP; done
```

You should saw the new services with new range.

To delete:

```bash
kubectl delete servicecidr newcidr1
```

## How to drain the worker node

This should be used when worker will become unavailable and its workload Pods should be moved safely to another node.

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force
```

---

## Kubernetes IPAM CIDR allocation

### Cluster pod CIDR

It is recommended to specify the required Cluster pod CIDR when kubeadm init.

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

It is accessible via kube-controller-manager

```bash
k describe po kube-controller-manager-k8s-t4-cp1 -n kube-system | grep cidr
--cluster-cidr=10.244.0.0/16
```

### Node pod CIDR

It can be change before joinning worker node. If joined, need to drain all the worker node.

```bash
sudo nano /etc/kubernetes/manifests/kube-controller-manager.yaml

// Add

- --node-cidr-mask-size=25
```

CIDR Mask means the total range a node given.

---

## References:

- [Extend Service IP Ranges | Kubernetes](https://kubernetes.io/docs/tasks/network/extend-service-ip-ranges/)
- [Container Runtimes | Kubernetes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)
- [Installing kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Creating a cluster with kubeadm | Kubernetes](https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [kubeadm join | Kubernetes](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join)
- [Ports and Protocols | Kubernetes](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
