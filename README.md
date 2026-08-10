# Kubernetes RBAC — User Authentication with TLS Certificates

This repository demonstrates how to create a Kubernetes user using **client certificates**, authenticate that user with the Kubernetes API server, and authorize the user using **RBAC**.

The complete flow is:

```text
User
 │
 │ 1. Generate private key
 ▼
mv.key
 │
 │ 2. Generate Certificate Signing Request
 ▼
mv.csr
 │
 │ 3. Create Kubernetes CertificateSigningRequest
 ▼
Kubernetes CSR
 │
 │ 4. Approve CSR
 ▼
Signed client certificate
 │
 ▼
mv.crt
 │
 │ 5. Configure kubeconfig
 ▼
User: mv
 │
 │ 6. Create ClusterRole
 ▼
ClusterRole
 │
 │ 7. Create ClusterRoleBinding
 ▼
mv ───────────────► Permissions
```

---

# 1. Objective

By completing this setup, the user `mv` will be able to authenticate to the Kubernetes cluster using a client certificate.

RBAC will then determine what `mv` is allowed to do.

For this example:

```text
Authentication:
    How do you prove you are mv?
    → Client certificate

Authorization:
    What is mv allowed to do?
    → RBAC

Identity:
    mv

Permission:
    Defined by ClusterRole

Binding:
    ClusterRoleBinding
```

---

# 2. Prerequisites

The following tools are required:

```bash
kubectl
openssl
base64
```

For local Kubernetes learning, this repository uses a `kind` cluster.

Check:

```bash
kubectl version --client
```

```bash
openssl version
```

Check the current Kubernetes context:

```bash
kubectl config current-context
```

---

# 3. Repository Structure

```text
24-RBAC/
│
├── README.md
│
├── 01-user/
│   ├── 01-generate-key.sh
│   └── 02-generate-csr.sh
│
├── 02-kubernetes-csr/
│   ├── csr.yaml
│   └── commands.sh
│
├── 03-rbac/
│   ├── cluster-role.yaml
│   └── cluster-role-binding.yaml
│
├── 04-kubeconfig/
│   └── commands.sh
│
└── .gitignore
```

---

# 4. Step 1 — Generate User Private Key

Move into the user directory:

```bash
cd 01-user
```

Generate a 2048-bit RSA private key:

```bash
openssl genrsa -out mv.key 2048
```

Verify:

```bash
ls -l mv.key
```

The private key is the user's secret credential.

```text
mv.key
   │
   └── MUST remain private
```

Do **not** commit this file to GitHub.

---

# 5. Step 2 — Generate Certificate Signing Request

Generate a CSR for user `mv`:

```bash
MSYS_NO_PATHCONV=1 openssl req -new \
  -key mv.key \
  -out mv.csr \
  -subj "/CN=mv"
```

The important part is:

```text
/CN=mv
```

Kubernetes will use the certificate's **Common Name (CN)** as the username.

Therefore:

```text
Certificate CN
      │
      ▼
     mv
      │
      ▼
Kubernetes username = mv
```

Verify the CSR:

```bash
openssl req -in mv.csr -noout -subject
```

Expected:

```text
subject=CN = mv
```

---

# 6. Step 3 — Convert CSR to Base64

Kubernetes `CertificateSigningRequest` expects the CSR in base64.

Run:

```bash
cat mv.csr | base64 | tr -d '\n'
```

Copy the complete output.

It will look similar to:

```text
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0...
```

This is **not encryption**.

It is simply base64 encoding.

The flow is:

```text
mv.csr
  │
  │ base64
  ▼
base64 encoded CSR
```

---

# 7. Step 4 — Create Kubernetes CertificateSigningRequest

Create:

```text
02-kubernetes-csr/csr.yaml
```

Example:

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest

metadata:
  name: mv

spec:
  request: <BASE64_ENCODED_CSR>

  signerName: kubernetes.io/kube-apiserver-client

  expirationSeconds: 864000

  usages:
    - client auth
```

Replace:

```text
<BASE64_ENCODED_CSR>
```

with the output generated previously.

Apply:

```bash
kubectl apply -f csr.yaml
```

Check:

```bash
kubectl get csr
```

Expected:

```text
NAME   AGE   SIGNERNAME                            REQUESTOR
mv     ...   kubernetes.io/kube-apiserver-client   kubernetes-admin
```

Initially:

```text
CONDITION
Pending
```

---

# 8. Step 5 — Approve the CSR

Approve:

```bash
kubectl certificate approve mv
```

Check:

```bash
kubectl get csr mv
```

Expected:

```text
CONDITION
Approved,Issued
```

This means Kubernetes has approved and signed the certificate.

---

# 9. Step 6 — Retrieve the Signed Certificate

The signed certificate is stored inside:

```text
.status.certificate
```

Check it:

```bash
kubectl get csr mv -o jsonpath='{.status.certificate}'
```

This value is base64 encoded.

Save it first:

```bash
kubectl get csr mv \
  -o jsonpath='{.status.certificate}' \
  > certificate.b64
```

Decode:

```bash
base64 -d certificate.b64 > mv.crt
```

Verify:

```bash
openssl x509 -in mv.crt -noout -subject -issuer -dates
```

Expected:

```text
subject=CN = mv
issuer=CN = kubernetes
notBefore=...
notAfter=...
```

The final authentication material is now:

```text
mv.key
   +
mv.crt
```

The private key proves possession of the identity.

---

# 10. Important Certificate Concept

There are three different things in this workflow:

```text
mv.key
   │
   └── Private key
       Never share

mv.csr
   │
   └── Certificate Signing Request
       Request for a certificate

mv.crt
   │
   └── Signed client certificate
       Used for authentication
```

And one additional representation:

```text
certificate.b64
```

This is simply the base64 representation of the certificate.

Base64 is **encoding**, not security.

---

# 11. Step 7 — Configure kubeconfig User

Add the user to kubeconfig:

```bash
kubectl config set-credentials mv \
  --client-key ./mv.key \
  --client-certificate ./mv.crt \
  --embed-certs=true
```

Verify:

```bash
kubectl config view
```

or:

```bash
kubectl config get-users
```

The kubeconfig now contains:

```text
User
 └── mv
      ├── client certificate
      └── client private key
```

---

# 12. Step 8 — Create a Context for mv

A context connects:

```text
Cluster + User + Namespace
```

Create the context:

```bash
kubectl config set-context mv \
  --cluster=kind-cluster3 \
  --user=mv
```

Check:

```bash
kubectl config get-contexts
```

You should see something similar to:

```text
NAME            CLUSTER          AUTHINFO
kind-cluster3   kind-cluster3    kind-cluster3
mv              kind-cluster3    mv
```

The important distinction is:

```text
Cluster
    ↓
kind-cluster3

User
    ↓
mv

Context
    ↓
mv + kind-cluster3
```

---

# 13. Step 9 — Switch to mv Context

```bash
kubectl config use-context mv
```

Check:

```bash
kubectl config current-context
```

Expected:

```text
mv
```

At this point kubectl is attempting to access the cluster as:

```text
User = mv
```

---

# 14. Step 10 — Verify Authentication

Run:

```bash
kubectl auth whoami
```

Expected:

```text
ATTRIBUTE   VALUE
Username    mv
```

Authentication is now working.

However, authentication does **not** automatically give permissions.

Try:

```bash
kubectl get pods
```

Depending on RBAC configuration, you may receive:

```text
Error from server (Forbidden):
pods is forbidden
```

This is expected.

The user has authenticated successfully but does not yet have authorization.

---

# 15. Authentication vs Authorization

This distinction is extremely important.

```text
AUTHENTICATION
      │
      │ "Who are you?"
      ▼
      mv
      │
      ▼
AUTHORIZATION
      │
      │ "What can mv do?"
      ▼
      RBAC
```

Certificate authentication establishes:

```text
mv = authenticated user
```

RBAC determines:

```text
mv = allowed actions
```

---

# 16. Step 11 — Create ClusterRole

Create:

```text
03-rbac/cluster-role.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

metadata:
  name: pod-reader

rules:
  - apiGroups:
      - ""

    resources:
      - pods

    verbs:
      - get
      - list
      - watch
```

Apply:

```bash
kubectl apply -f cluster-role.yaml
```

Check:

```bash
kubectl get clusterrole pod-reader
```

---

# 17. Step 12 — Create ClusterRoleBinding

Create:

```text
03-rbac/cluster-role-binding.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: mv-pod-reader

subjects:
  - kind: User
    name: mv
    apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f cluster-role-binding.yaml
```

---

# 18. Step 13 — Verify RBAC

Switch to the `mv` context:

```bash
kubectl config use-context mv
```

Check:

```bash
kubectl auth whoami
```

Expected:

```text
Username: mv
```

Check authorization:

```bash
kubectl auth can-i get pods
```

Expected:

```text
yes
```

Check:

```bash
kubectl auth can-i list pods
```

Expected:

```text
yes
```

Check an action that wasn't granted:

```bash
kubectl auth can-i delete pods
```

Expected:

```text
no
```

---

# 19. Test the User

Try:

```bash
kubectl get pods
```

The user should now be able to list pods.

Try:

```bash
kubectl delete pod <pod-name>
```

This should fail because `delete` was not included in the ClusterRole.

This demonstrates RBAC working.

---

# 20. Complete Architecture

The complete flow is:

```text
                    Kubernetes Cluster
                           │
                           ▼
                    Kubernetes API
                         Server
                           ▲
                           │
                    TLS Authentication
                           │
                           │ mv.crt
                           │
                     ┌─────┴─────┐
                     │    User   │
                     │     mv    │
                     └─────┬─────┘
                           │
                     kubeconfig
                           │
                    ┌──────┴──────┐
                    │   Context   │
                    │             │
                    │ mv          │
                    │     +       │
                    │ kind-cluster3
                    └──────┬──────┘
                           │
                           ▼
                    Authentication
                           │
                           ▼
                    User = mv
                           │
                           ▼
                     RBAC Engine
                           │
                    ┌──────┴──────┐
                    │ ClusterRole │
                    │ pod-reader  │
                    └──────┬──────┘
                           │
                           ▼
                 ClusterRoleBinding
                           │
                           ▼
                    User = mv
                           │
                           ▼
                   Allowed Actions
```

---

# 21. The Mental Model

Remember these five concepts:

```text
1. KEY
   Who can prove possession?

2. CERTIFICATE
   Who does Kubernetes recognize you as?

3. KUBECONFIG
   How does kubectl use your identity?

4. RBAC
   What are you allowed to do?

5. BINDING
   Who receives those permissions?
```

In this example:

```text
mv.key
   ↓
mv.crt
   ↓
kubeconfig user: mv
   ↓
authentication
   ↓
User: mv
   ↓
ClusterRole: pod-reader
   ↓
ClusterRoleBinding
   ↓
Permissions
```

---

# 22. Common Debugging Commands

Check current context:

```bash
kubectl config current-context
```

List contexts:

```bash
kubectl config get-contexts
```

List users:

```bash
kubectl config get-users
```

View kubeconfig:

```bash
kubectl config view
```

Check identity:

```bash
kubectl auth whoami
```

Check permission:

```bash
kubectl auth can-i get pods
```

Check permission as another user:

```bash
kubectl auth can-i get pods --as=mv
```

Check CSR:

```bash
kubectl get csr
```

Inspect CSR:

```bash
kubectl get csr mv -o yaml
```

Check certificate:

```bash
openssl x509 -in mv.crt -noout -subject -issuer -dates
```

---

# 23. Important Security Rules

Never commit these files:

```text
*.key
*.crt
*.pem
certificate.b64
```

Especially:

```text
mv.key
```

The private key must remain private.

The following can generally be committed:

```text
README.md
csr.yaml
cluster-role.yaml
cluster-role-binding.yaml
commands.sh
```

But make sure `csr.yaml` does not contain a real sensitive credential if you are publishing the repository.

---

# 24. .gitignore

Recommended:

```gitignore
# Private keys
*.key
*.pem

# Certificates
*.crt

# Base64 encoded certificate/credential files
*.b64

# Local kubeconfig
kubeconfig

# Temporary files
*.tmp
*.log
```

---

# 25. Cleanup

When finished, switch back to the administrative context:

```bash
kubectl config use-context kind-cluster3
```

Delete the RBAC resources:

```bash
kubectl delete clusterrolebinding mv-pod-reader
```

```bash
kubectl delete clusterrole pod-reader
```

Delete the Kubernetes CSR:

```bash
kubectl delete csr mv
```

Remove the kubeconfig user:

```bash
kubectl config delete-user mv
```

Remove the context:

```bash
kubectl config delete-context mv
```

---

# 26. Final Learning Model

The entire exercise demonstrates:

```text
                 ┌──────────────┐
                 │    mv.key    │
                 └──────┬───────┘
                        │
                   Generate CSR
                        │
                        ▼
                 ┌──────────────┐
                 │    mv.csr    │
                 └──────┬───────┘
                        │
                  Base64 encode
                        │
                        ▼
              Kubernetes CSR object
                        │
                  Approve + Sign
                        │
                        ▼
                 ┌──────────────┐
                 │    mv.crt    │
                 └──────┬───────┘
                        │
                 kubeconfig user
                        │
                        ▼
                 ┌──────────────┐
                 │      mv      │
                 └──────┬───────┘
                        │
                  Authenticate
                        │
                        ▼
                 ┌──────────────┐
                 │     RBAC     │
                 └──────┬───────┘
                        │
                 ClusterRole
                        │
                        ▼
               ClusterRoleBinding
                        │
                        ▼
                 Permissions
```

The key distinction to remember:

> **Certificate answers "Who are you?"**
> **RBAC answers "What are you allowed to do?"**
