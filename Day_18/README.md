# Single-Cluster LLM Inference Architecture

![Single-Cluster LLM Inference Architecture](assets/Single-Cluster%20LLM%20Inference%20Architecture.png)

## Prerequisites for clean VM

Before starting this lab, prepare a Kubernetes cluster with GPU support and the required client tools.

### Kubernetes cluster

- A working kubeadm Kubernetes cluster.
- At least one control-plane node.
- At least one GPU-capable worker node.
- A working CNI plugin such as Cilium.
- kubectl configured to access the cluster.

Verify:

```bash
kubectl get nodes
kubectl get pods -A
```

All required nodes should report:

```
Ready
```

### NVIDIA GPU support

The GPU worker must have:

- NVIDIA driver installed.
- NVIDIA Container Toolkit configured for the container runtime.
- NVIDIA Kubernetes Device Plugin installed.

Alternatively, NVIDIA GPU Operator can be used to manage the GPU software stack.

Verify the GPU on the host:

```bash
nvidia-smi
```

Then verify that Kubernetes advertises the GPU:

```bash
kubectl describe node <gpu-worker> | grep -A5 nvidia.com/gpu
```

or:

```bash
kubectl get node <gpu-worker> \
  -o jsonpath='{.status.allocatable.nvidia\.com/gpu}'
```

For a single-GPU worker, the expected result is:

```
1
```

A simple GPU test pod should also successfully access the GPU before deploying vLLM.

### Required command-line tools

Install and verify:

```bash
kubectl version --client
helm version
istioctl version --remote=false
jq --version
git --version
curl --version
hey --version
```

The tools are used for:

| Tool | Purpose |
| --- | --- |
| kubectl | Manage and inspect Kubernetes resources |
| helm | Install KEDA, Prometheus, and llm-d components |
| istioctl | Install Istio with Gateway API Inference Extension support |
| jq | Process JSON output during llm-d setup and testing |
| git | Clone the llm-d repository |
| curl | Test the OpenAI-compatible inference endpoint |
| hey | Generate concurrent inference traffic for scaling tests |

## Nvidia GPU time-slicing

To host multiple LLM in GPU, GPU does not behave like CPU to able to operate many POD at the same time. To achieve this, we will be using time-slicing by NVIDIA Device Plugin.

### Install Nvidia Device Plugin

```bash
helm repo add nvdp \
  https://nvidia.github.io/k8s-device-plugin
helm repo update
helm upgrade --install nvdp \
  nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --create-namespace \
  -f nvidia-device-plugin-values.yaml
```

## Create namespace

```bash
k create ns inference
```

## Install KEDA by helm

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm upgrade --install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --wait
```

## Install prometheus by helm

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update

helm upgrade --install prometheus \
  prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-values.yaml

helm upgrade --install prometheus \
  prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-values.yaml
```

## Secret Generate Command

```bash
kubectl create secret generic hf-token \
  -n inference \
  --from-literal=token='hf_your_token_here' \
  --dry-run=client \
  -o yaml > hf-token-secret.yaml

kubectl apply -f hf-token-secret.yaml
```

## Install Gateway API +  Inference Estension CRDs

```bash
GATEWAY_API_VERSION=v1.5.1
GAIE_VERSION=v1.5.0

kubectl apply -f \
https://github.com/kubernetes-sigs/gateway-api/releases/download/${GATEWAY_API_VERSION}/standard-install.yaml

kubectl apply -f \
https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/${GAIE_VERSION}/v1-manifests.yaml
```

### Verify

```bash
kubectl api-resources \
  --api-group=gateway.networking.k8s.io
kubectl api-resources \
  --api-group=inference.networking.k8s.io
```

## Install Istio

Istio is the gateway controller. llm-d officially supports Istio, Agentgateway, Envoy AI Gateway and GKE Gateway.

```bash
ISTIO_VERSION=1.29.2

curl -L https://istio.io/downloadIstio \
  | ISTIO_VERSION=${ISTIO_VERSION} sh -

export PATH="$PWD/istio-${ISTIO_VERSION}/bin:$PATH"

istioctl install -y \
  --set values.pilot.env.ENABLE_GATEWAY_API_INFERENCE_EXTENSION=true
```

### Verify

```bash
kubectl get pods -n istio-system
```

## Create Inference Gateway

```bash
LLM_D_VERSION=v0.9.0
kubectl apply -k \
"https://github.com/llm-d/llm-d/guides/recipes/gateway/istio?ref=${LLM_D_VERSION}" \
-n inference
```

### Verify

```bash
kubectl get gateway -n inference
```

```
NAME                      CLASS   PROGRAMMED
llm-d-inference-gateway   istio   True
```

## Install llm-d Router

```bash
git clone \
  --branch v0.9.0 \
  --depth 1 \
  https://github.com/llm-d/llm-d.git

cd llm-d

source guides/env.sh
```

### Verify

```bash
echo $ROUTER_GATEWAY_CHART
echo $ROUTER_CHART_VERSION
helm install gemma-router \
  "${ROUTER_GATEWAY_CHART}" \
  -f guides/recipes/router/base.values.yaml \
  -f ~/projects/Kubernetes-Learning-Path/Day_18/llm-d-gemma-router-values.yaml \
  -n inference \
  --version "${ROUTER_CHART_VERSION}"
```

## Test

### Prometheus

```bash
kubectl port-forward \
  -n monitoring \
  svc/prometheus-server \
  9090:9090
```

http://localhost:9090

try vllm:num_requests_waiting

### KEDA

```bash
# 2,000 requests, 80 concurrent — at 16, vLLM absorbs everything into the running batch and no queue forms
hey -n 2000 -c 80 -m POST \
  -H "Content-Type: application/json" \
  -d '{"model":"google/gemma-3-270m-it","messages":[{"role":"user","content":"Summarize the CAP theorem."}],"max_tokens":1024}' \
  http://localhost:8000/v1/chat/completions
```

Another terminal:

```bash
watch -n 2 kubectl get scaledobject,hpa,pods -n inference
```

Command to test the system:

```bash
kubectl port-forward \
  -n inference \
  svc/llm-d-inference-gateway-istio \
  8080:80
```

```bash
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "google/gemma-3-270m-it",
    "messages": [
      {
        "role": "user",
        "content": "Explain Kubernetes in one sentence."
      }
    ],
    "max_tokens": 50
  }'
```

## Reference

- https://scaleops.com/blog/vllm-kubernetes/
- https://aws.amazon.com/blogs/machine-learning/introducing-disaggregated-inference-on-aws-powered-by-llm-d/
- https://llm-d.ai/docs/getting-started/quickstart

Hardware used: Nvidia 4060 Ti
