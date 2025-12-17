# Kubernetes RBAC – Dev Namespace Admin (Certificate-Based User)

### Overview
This project demonstrates how to create a **certificate-based Kubernetes user**
with **full administrative access restricted to a single namespace (`dev`)**.
The setup follows real-world RBAC and least-privilege best practices.

### What is CSR?
**CSR (Certificate Signing Request)** is a request sent to the Kubernetes API
to obtain a signed client certificate.

In this project:
- We generates a private key (`dev.key`)
- A CSR (`dev.csr`) is created with identity `CN=dev-user`
- Kubernetes signs the CSR after approval
- The signed certificate (`dev.crt`) is used for authentication

> Kubernetes identifies users using the `Common Name (CN)` field of the certificate,
> which must match the RBAC user name.

---

### Why This Project?
- Understand **Kubernetes authentication** using x509 certificates
- Implement **RBAC with least privilege**
- Prevent accidental changes to **system namespaces**
- Practice **production-like Kubernetes security patterns**


### Components
- **Namespace**: `dev`
- **User**: `dev-user` (x509 certificate-based)
- **Role**: `dev-admin-role` (full access within `dev`)
- **RoleBinding**: Binds `dev-user` to the role
- **Context**: `dev-context`


### Access Flow
```txt
User → Certificate → Role → RoleBinding → Context → Namespace
```

### Deployment steps

**1. Create Namespace:**
Create the `dev` namespace using the provided manifest:
```sh
kubectl apply -f namespace.yaml
```

**2. Generate Certificates for `dev-user`**
```sh
# Generate private key for dev-user
openssl genrsa -out dev.key 2048

# Create CSR (CN = username in Kubernetes)
openssl req -new -key dev.key -out dev.csr -subj "/CN=dev-user"

# Base64 encode CSR
cat dev.csr | base64 | tr -d '\n'
```

**3. Create Kubernetes CSR Object:**
Create the CSR resource using the provided YAML file:
```sh
kubectl apply -f dev-user-csr.yaml
```

**4. Approve CSR & Extract Client Certificate**
```sh
# Approve the CSR
kubectl certificate approve dev-user

# Extract signed certificate
kubectl get csr dev-user \
  -o jsonpath='{.status.certificate}' | base64 --decode > dev.crt
```

**5. Create Role & RoleBinding:**
Apply RBAC manifests to grant full admin access only within `dev namespace`:
```sh
kubectl apply -f rbac/role-dev-admin.yaml
kubectl apply -f rbac/rolebinding-dev-admin.yaml
```

**6. Configure kubeconfig Context**
```sh
# Register dev-user credentials
kubectl config set-credentials dev-user \
  --client-certificate=dev.crt \
  --client-key=dev.key

# Create a context scoped to dev namespace
kubectl config set-context dev-context \
  --cluster=kind-dev-cluster \
  --user=dev-user \
  --namespace=dev

# Switch to dev context
kubectl config use-context dev-context
```

**7. Verification**
```sh
kubectl auth can-i create pods -n dev
# yes

kubectl auth can-i delete deployments -n dev
# yes

kubectl auth can-i get pods -n kube-system
# no
```


### Security Notes
- User has no access to `kube-system`
- Permissions are namespace-scoped
- `ClusterRole` is intentionally avoided
- Follows least-privilege and blast-radius reduction principles


### Use Cases
- Multi-tenant clusters
- Developer-only namespace access
- Safe local / lab Kubernetes environments
- RBAC demonstration for learning & practice