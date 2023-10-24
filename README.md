# Vault Secrets Operator

These are my notes while going through the [Vault Secrets Operator guide](https://developer.hashicorp.com/vault/tutorials/kubernetes/vault-secrets-operator).

## Overview

The Vault Secrets Operator operates by watching for changes to its supported set of Custom Resource Definitions (CRD). Each CRD provides the specification required to allow the operator to synchronize from one of the supported sources for secrets to a Kubernetes Secret. The operator writes the source secret data directly to the destination Kubernetes Secret, ensuring that any changes made to the source are replicated to the destination over its lifetime. In this way, an application only needs to have access to the destination secret in order to make use of the secret data contained within.

## Configure Kubernetes

In Kubernetes, a service account provides an identity for processes that run in a Pod so that the processes can contact the API server. To configure Kubernetes, open the provided `vault-auth-service-account.yaml` file in your preferred text editor and examine its content for the service account definition to be used for this tutorial.

```bash
kubectl create ns app
kubectl -n app apply -f k8s/vault-auth-service-account.yaml
```

### Kubernetes 1.24+ only

The service account generates a secret that is required for configuration automatically in Kubernetes 1.23. In Kubernetes 1.24+, you need to create the secret explicitly. You can use `kubectl get nodes` to check your Kubernetes version.

```bash
kubectl -n app apply -f k8s/vault-auth-secret.yaml
```

Now, retrieve the new secret name and token and store them as environment variables: `SA_SECRET_NAME`, `SA_JWT_TOKEN`, `SA_CA_CRT`, and `K8S_HOST`.

```bash
export SA_SECRET_NAME=$(kubectl -n app get secrets --output=json \
    | jq -r '.items[].metadata | select(.name|startswith("vault-auth-")).name')
export SA_JWT_TOKEN=$(kubectl -n app get secret $SA_SECRET_NAME \
    --output 'go-template={{ .data.token }}' | base64 --decode)
export SA_CA_CRT=$(kubectl -n app config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
export K8S_HOST=$(kubectl -n app config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.server}')
```

## Configure Vault

Now, connect to Vault, enable and configure Kubernetes authentication, KV secrets engine, a role and policy for Kubernetes, and create a static secret.

```bash
# Login To Vault
export VAULT_ADDR='http://127.0.0.1:8200'
vault login

# Create a policy
vault policy write myapp-kv-ro - <<EOF
path "kvv2/*" {
    capabilities = ["read", "list"]
}
EOF

# Enable k8s auth
vault auth enable -path demo-auth-mount kubernetes

# Configure the auth
vault write auth/demo-auth-mount/config \
     token_reviewer_jwt="$SA_JWT_TOKEN" \
     kubernetes_host="$K8S_HOST" \
     kubernetes_ca_cert="$SA_CA_CRT" \
     issuer="https://kubernetes.default.svc.cluster.local"

# Create a role
vault write auth/demo-auth-mount/role/example \
     bound_service_account_names=vault-auth \
     bound_service_account_namespaces=app \
     token_policies=myapp-kv-ro \
     ttl=24h
```

## Testing Kubernetes Authentication

Here, we are going to test that authentication works using a simple app with the image `burtlo/devwebapp-ruby:k8s`. You can skip this step if desired.

```bash
export VAULT_ADDR='http://127.0.0.1:8200'

kubectl -n app apply -f auth-test/devwebapp.yaml

kubectl -n app get pods

kubectl -n app exec -ti devwebapp -- /bin/sh
```

Set `KUBE_TOKEN` to the service account token inside the `devwebapp` and authenticate with Vault through the example role with the `KUBE_TOKEN`. **Remember to use the external IP address for Vault here.**

```bash
export KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
export VAULT_ADDR='http://192.168.0.238:8200'

curl --request POST --data '{"jwt": "'"$KUBE_TOKEN"'", "role": "example"}' $VAULT_ADDR/v1/auth/demo-auth-mount/login
```

Clean up the temporary container.

```bash
kubectl -n app delete -f auth-test/devwebapp.yaml
```

## Configure the Secrets in Vault

```bash
vault secrets enable -path=kvv2 kv-v2
vault kv put kvv2/webapp/config username="static-user" password="static-password"
```

## Deploy the Vault Secrets Operator

Use Helm to deploy the Vault Secrets Operator. **You need to update the `vault/vault-operator-values.yaml` file to contain the correct address for Vault**.

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault-secrets-operator hashicorp/vault-secrets-operator --version 0.3.1 -n vault-secrets-operator-system --create-namespace --values vault/vault-operator-values.yaml
```

## Deploy and Sync a Secret

```bash
kubectl create ns app
kubectl apply -f vault/vault-auth-static.yaml
kubectl apply -f vault/static-secret.yaml
```

## Test the Secret

```bash
kubectl -n app get secret secretkv -o jsonpath='{.data.username}' | base64 -d
kubectl -n app get secret secretkv -o jsonpath='{.data.password}' | base64 -d
```

## Rotate the Secret and Get it from Kubernetes Again

```bash
vault kv put kvv2/webapp/config username="static-user2" password="static-password2"
# We need to wait a little for this to update
sleep 20

# It should be ok after about 20 seconds
kubectl -n app get secret secretkv -o jsonpath='{.data.username}' | base64 -d
kubectl -n app get secret secretkv -o jsonpath='{.data.password}' | base64 -d
```

### Clean Up

If you want to clean up the secrets, use the following commands:

```bash
kubectl delete -f vault/vault-auth-static.yaml
kubectl delete -f vault/static-secret.yaml
```