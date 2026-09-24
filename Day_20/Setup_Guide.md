# Gateway API + cert-manager Certificate Automation Lab

## Objective:
This lab focuses only on how **cert-manager integrates with Kubernetes Gateway API** to automatically issue and manage a TLS certificate. 

Cert-manager is a Kubernetes certificate management system, which is a **CNCF Graduated Project**. It use CRDs and controllers to automate the certificate lifecycle, including certificate issuance, renewal, and storage in Kubernetes Secrets. 

When intergrated with Gateway API, cert-manager can detect **annotation** configured in Gateway API, and it will create a certificate and a temporary certificateRequest for clusterissuer/issuer to sign the certificate. Eventually, certificate is updated and stored into the secret with name specified by Gateway API's references.

## Cert-manager workflow:
1. **Notice request**: Cert-manager detects a need from Gateway API via annotation.
2. **Create certificate**: Cert-manager create a certificate resource, storing dnsNames, issuerRef and secretName.
3. **Create certificateRequest**: cert-manager generates the private key and creates a CertificateRequest containing a CSR and sent to issuer/clusterisser.
4. **Signs via CA**: Issuer/ClusterIssuer signs the certificateRequest and return issued certificate to cert-manager.
5. **Update Certificate**: cert-manager update certificate status and ready.
6. **Store in secret**: Cert-manager will store the issued key-pairs in Kubernetes secret with name defined in Gateway API's certificateRefs.
7. **Listen updated secret**: Gateway API will continuous listening to the secret. Once secret is valid, Gateway API is ready for TLS termination.

# Stage 1 — Install Gateway API

## Step 1 — Install Gateway API CRDs


```bash
kubectl kustomize \
  "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.7.2" \
  | kubectl apply -f -
```

Validate:

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

You should see resources such as:

```text
gatewayclasses.gateway.networking.k8s.io
gateways.gateway.networking.k8s.io
httproutes.gateway.networking.k8s.io
referencegrants.gateway.networking.k8s.io
```

# Stage 2 — Install cert-manager

## Step 2 — Create the cert-manager namespace

```bash
kubectl create namespace cert-manager
```


## Step 3 — Install cert-manager with Gateway API support

```bash
helm upgrade --install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --set crds.enabled=true \
  --set config.gatewayAPI.enabled=true
```

`crds.enabled=true` installs cert-manager resources such as:

```text
Certificate
CertificateRequest
Issuer
ClusterIssuer
```

`config.gatewayAPI.enabled=true` allows cert-manager to watch Gateway API resources.

To validate:

```bash
kubectl get pods -n cert-manager
```

Check the CRDs:

```bash
kubectl get crd | grep cert-manager
```


# Stage 3 — Create the CA and ClusterIssuer

## Step 4 — Generate a Root CA
According to documentation, CA issuer represents a Certificate Authority whose certificate and private key are stored inside the cluster as a Kubernetes Secret. It is commonly used by private PKI environments. Learning CA issuer type as fundamental helps because it demonstrates basic certificate flow clearly without complex configuration.

Create a directory:

```bash
mkdir -p ~/ca
```

Generate the private key:

```bash
openssl genrsa -out ~/ca/ca.key 4096
```

Generate the self-signed root CA certificate:

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

The result is:

```text
~/ca/ca.key
~/ca/ca.crt
```

> You can able to view it by the command below

```bash
openssl x509 -in ~/ca/ca.crt -noout -text -subject -issuer -dates
openssl rsa -in ~/ca/ca.key -noout -text 
```

## Step 5 — Store the CA in Kubernetes

Once the CA key pair is done, create a Secret containing the CA certificate and private key:

```bash
kubectl create secret tls ca-key-pair \
  --cert="$HOME/ca/ca.crt" \
  --key="$HOME/ca/ca.key" \
  -n cert-manager
```

To validate:

```bash
kubectl get secret ca-key-pair -n cert-manager
```

## Step 6 — Create the ClusterIssuer

For this lab, we are using clusterIssuer. The main difference between issuer and clusterIssuer is cluster issuer is cluster-wide while isser is namespace wide. For issuer, it must be in same namespace with Gateway API. 

Let's create `clusterissuer.yaml` by `vim clusterissuer.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: ca-key-pair
```

Apply:

```bash
kubectl apply -f clusterissuer.yaml
```

Validate:

```bash
kubectl get clusterissuer
```

Expected:

```text
NAME        READY
ca-issuer   True
```

Inspect if necessary:

```bash
kubectl describe clusterissuer ca-issuer
```

The relationship is:

```text
ClusterIssuer: ca-issuer
        │
        │ uses
        ▼
Secret: ca-key-pair
        │
        ├── tls.crt
        └── tls.key
```

---

# Stage 4 — Monitor cert-manager

## Step 7 — Watch cert-manager Controller Logs

Before creating the Gateway, start monitoring cert-manager:

```bash
kubectl logs -n cert-manager deploy/cert-manager -f
```

Keep this terminal running and you may highlight the latest log as a marking.

The cert-manager controller watches Kubernetes resources and reacts when a new annotated Gateway requests a certificate.

Open a new terminal for executing the following step to see the logs changes.

---

# Stage 5 — Create an Annotated Gateway

## Step 8 — Create the Gateway

Create `gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gatewaya
  annotations:
    cert-manager.io/cluster-issuer: ca-issuer

spec:
  gatewayClassName: nginx

  listeners:
    - name: https
      hostname: app.lab.local
      port: 443
      protocol: HTTPS

      tls:
        mode: Terminate
        certificateRefs:
          - name: local-tls
            kind: Secret
            group: ""
```

The important parts for cert-manager are:

```yaml
annotations:
  cert-manager.io/cluster-issuer: ca-issuer
```

This tells cert-manager:

```text
Use ClusterIssuer "ca-issuer"
to issue the Gateway certificate.
```

The listener defines:

```yaml
hostname: app.lab.local
```

which becomes the DNS name requested in the certificate.

The Gateway also specifies:

```yaml
certificateRefs:
  - name: local-tls
```

which tells cert-manager where the resulting certificate should be stored.

---

## Step 9 — Apply the Gateway

```bash
kubectl apply -f gateway.yaml
```

Check:

```bash
kubectl get gateway
```

At the same time, observe the cert-manager logs from the other terminal.

You should see cert-manager detecting the Gateway and beginning the certificate issuance process.

---

# Stage 6 — Observe cert-manager Resources

## Step 10 — Check the Certificate

```bash
kubectl get certificate -o wide
```

Expected:

```text
NAME         READY   SECRET       ISSUER      STATUS                                          AGE
local-tls    True    local-tls    ca-issuer   Certificate is up to date and has not expired   1m
```

Inspect:

```bash
kubectl describe certificate local-tls
```

The `Certificate` represents the desired certificate.

It tells cert-manager:

```text
I need a certificate for app.lab.local
and the result should be stored in local-tls.
```

## Step 11 — Check the CertificateRequest

```bash
kubectl get certificaterequest -o wide
```

Example:

```text
NAME           APPROVED   DENIED   READY   ISSUER      REQUESTER                                         STATUS                                         AGE
local-tls-1    True                True    ca-issuer   system:serviceaccount:cert-manager:cert-manager   Certificate fetched from issuer successfully   1m
```

Inspect to see the context:

```bash
kubectl describe certificaterequest local-tls-1
```
It contains:
Request: Certificate Signing Request [CSR].
CA: CA cert used to validify the issuer is real.
Certificate: the issued certificate.


## Step 12 — Check the Generated TLS Secret

```bash
kubectl get secret local-tls
```

Inspect:

```bash
kubectl describe secret local-tls
```

You should see keys such as:

```text
tls.crt
tls.key
ca.crt
```

Their purposes are:

```text
tls.crt
└── Signed certificate for app.lab.local

tls.key
└── Private key belonging to the certificate

ca.crt
└── CA certificate that signed the certificate
```


# Congratulations 🥳

You had finished the lab. The entire lab demonstrates one core cert-manager workflow:

```text
Gateway
  │
  │ annotation:
  │ cert-manager.io/cluster-issuer: ca-issuer
  ▼
cert-manager controller
  │
  │ automatically creates
  ▼
Certificate
  │
  │ automatically creates
  ▼
CertificateRequest
  │
  │ sent to
  ▼
ClusterIssuer
  │
  │ uses
  ▼
CA Secret
  │
  │ signs CSR
  ▼
Issued Certificate
  │
  │ stored in
  ▼
Secret: local-tls
```

The three main resources to understand are:

```text
Certificate
```

The **desired certificate** managed by cert-manager.

```text
CertificateRequest
```

The **actual certificate signing request** sent to the issuer.

```text
Secret
```

The **final output** containing the issued certificate and private key.

So the core idea is:

```text
Gateway asks for TLS
        ↓
Certificate describes what is needed
        ↓
CertificateRequest asks the issuer to sign it
        ↓
Issuer signs it
        ↓
Secret stores the result
```
See you in next lab!
