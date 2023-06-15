# Kubernetes integration with external Vault
in this tuto we install and deploy vault as service in ubuntu vm and integrate it to kubernetes with an agent.
## Quick Start
- Install Vault 
- Deploy Vault 
- Configure Vault 
- Configuration k8s 
- enable and configure vault kuberntes authentications
- Install Vault Agent Injector and Test Configuration

## version

**Ubuntu**

22.04.1 LTS (GNU/Linux 5.15.0-72-generic x86_64)

**Vault**

v1.13.2 (b9b773f1628260423e6cc9745531fd903cae853f), built 2023-04-25T13:02:50Z;

**kubernetes**

Server Version: v1.26.4+k3s1

**vagrant agent**

hashicorp/vault-k8s:1.2.1

## Install Vault [link](https://developer.hashicorp.com/vault/downloads)

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault
```

## Deploy Vault [link](https://devopstales.github.io/kubernetes/k8s-vault-v2/)

Create file to persist vault data.

```bash
mkdir -p ./vault/data
```
create vault configuration file.
vi /etc/vault.d/vault.hcl

```bash
storage "raft" {
  path    = "./vault/data"
  node_id = "node1"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = "true"
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8200"
ui = true
```
Enable vault and start vault.
```bash
systemctl start vault
systemctl enable vault
```

## Configure Vault [link1](https://devopstales.github.io/kubernetes/k8s-vault-v2/) | [link2](https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-deploy)

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```
To initialize Vault use

```bash
vault operator init
```
To unseal the Vault, you must have the threshold number of unseal keys
```bash
vault operator unseal x3
```
Enables a secrets engine.
```bash
vault secrets enable kv
```
create secrets in kv/secret/devwebapp/config

```bash
vault kv put kv/secret/devwebapp/config username='giraffe' password='salsa'

vault kv get -format=json kv/secret/devwebapp/config | jq ".data"
```
create policy to access the secrets 

```bash
vault policy write devwebapp-kv-ro - <<EOF
path "kv/secret/devwebapp/*" {
    capabilities = ["read", "list"]
}
```

## Configuration k8s 

1. create Kubernetes ServiceAccount

```bash
cat > internal-app.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: devwebapp
---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: role-tokenreview-binding
    namespace: default
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:auth-delegator
  subjects:
  - kind: ServiceAccount
    name: devwebapp
    namespace: default
EOF

kubectl apply --filename internal-app.yaml
```
2. Kubernetes 1.24+ only: The name of the mountable secret is displayed in Kubernetes 1.23. In Kubernetes 1.24+, the token is not created automatically, and we must create it explicitly.

```bash
cat > vault-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: vault-token-g955r
  annotations:
    kubernetes.io/service-account.name: devwebapp
type: kubernetes.io/service-account-token
EOF

kubectl apply -f vault-secret.yaml
```
3. export variables 

```bash
export EXTERNAL_VAULT_ADDR=127.0.0.1
export K8S_HOST=127.0.0.1
export VAULT_SA_NAME=devwebapp
export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME \
    -o jsonpath="{.data.token}" | base64 --decode; echo)

export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME \
    -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)

```
## enable and configure vault kuberntes authentications

```bash

vault auth enable kubernetes

vault write auth/kubernetes/config \
  issuer="https://kubernetes.default.svc.cluster.local" \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="https://$K8S_HOST:6443" \
  kubernetes_ca_cert="$SA_CA_CRT"

vault write auth/kubernetes/role/devwebapp \
        bound_service_account_names=devwebapp \
        bound_service_account_namespaces=default \
        policies=devwebapp-kv-ro \
        ttl=24h
```

## Install Vault Agent Injector and Test Configuration

1. install helm
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```
2. install vault agent
```bash
helm install vault hashicorp/vault \
    --set "injector.externalVaultAddr=http://$EXTERNAL_VAULT_ADDR:8200"

```
3. deploy app
```bash
cat > devwebapp.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orgchart
  labels:
    app: orgchart
spec:
  selector:
    matchLabels:
      app: orgchart
  replicas: 1
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "devwebapp"
        vault.hashicorp.com/agent-inject-secret-config.txt: "kv/secret/devwebapp/config"
      labels:
        app: orgchart
    spec:
      serviceAccountName: devwebapp
      containers:
        - name: orgchart
          image: jweissig/app:0.0.1
EOF

kubectl apply -f devwebapp.yaml
```
4. test configuration 
```bash
kubectl exec \
    $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") \
    --container orgchart -- cat /vault/secrets/config.txt
```




