# Gateway API Cert manager lab setup

## Install gateway api:

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.7.2" | kubectl apply -f -
```

> Validate: kubectl get crd | grep gateway.networking.k8s.io

---

## install certmanager with gatewayapi enabled

create cert-manager ns

> Crd enabled is not in documentation, but be remind that is needed because it need to detect crd created by us [clusterissuer]

```bash
helm upgrade --install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --set crds.enabled=true \
  --set config.gatewayAPI.enabled=true
```

---

## Setup nginx gateway fabric

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway
```

> Validate: helm list -A

---

## Generate CA key pair

```bash
mkdir -p ~/ca
```

```text
// create folder
```

```text
// generates 4096-bit RSA private key
```

```bash
openssl genrsa -out ~/ca/ca.key 4096
```

Creates a self-signed root CA certificate valid for 10 years.

```text
// by default 30 days
```

```bash
openssl req -x509 -new -nodes \
  -key ~/ca/ca.key \
  -sha256 \
  -days 3650 \
  -out ~/ca/ca.crt \
  
  -subj "/CN=lab-root-ca" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"
```

> Validation command: openssl x509 -in ~/ca/ca.crt -noout -subject -dates -ext basicConstraints,keyUsage

> Standard practice for validity periods

> Root CA: 10-25 years

> Intermediate CA: 3-10 years

> Publis TLS leaf: 200 days

> Internal TLS/ mTLS: 30 days to 1 year

---

## Create secret for CA key pair

```bash
kubectl create secret tls ca-key-pair \
  --cert=$HOME/ca/ca.crt \
  --key=$HOME/ca/ca.key \
  -n cert-manager
```

---

## Create cluster issuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: ca-key-pair
```

---

## Inspect Certmanager

```text
k logs -f, then highlight the latest log
```

---

## Setup gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gatewaya
  annotations:
      cert-manager.io/issuer: ca-issuer
spec:
  gatewayClassName: nginx
  listeners:
  - protocol: HTTPS
    hostname: app.lab.local
    port: 80
    name: https
    allowedRoutes:
      namespaces:
        from: Same
    tls:
      mode: Terminate
      certificateRefs:
        - name: local-tls
          kind: Secret
          group: ""
```

---

## Expose NodePort for gatewayAPI for demonstration

```bash
helm upgrade ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric -n nginx-gateway \
  --reuse-values --set nginx.service.type=NodePort
```

---

## Create deployment with test pod. hashicorp/http-echo is an image that will reply on any fixed string

---

## setup httproute

---

## add dns record // in practise is external ip

```bash
echo "<NODE_IP> app.lab.local" | sudo tee -a /etc/hosts
```

---

## curl by app.lab.local:30870

---

[Install NGINX Gateway Fabric with Helm | NGINX Documentation](https://docs.nginx.com/nginx-gateway-fabric/install/helm/)

[Annotated Gateway resource - cert-manager Documentation](https://cert-manager.io/docs/usage/gateway/?utm_source=chatgpt.com)
