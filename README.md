# Vault-Secrets-Operator
My notes while going through the [Vault Secrets Operator guide](https://developer.hashicorp.com/vault/tutorials/kubernetes/vault-secrets-operator)

## Overview
The Vault Secrets Operator operates by watching for changes to its supported set of Custom Resource Definitions (CRD). Each CRD provides the specification required to allow the operator to synchronize from one of the supported sources for secrets to a Kubernetes Secret. The operator writes the source secret data directly to the destination Kubernetes Secret, ensuring that any changes made to the source are replicated to the destination over its lifetime. In this way, an application only needs to have access to the destination secret in order to make use of the secret data contained within.

## Configure K8s

In Kubernetes, a service account provides an identity for processes that run in a Pod so that the processes can contact the API server. Open the provided vault-auth-service-account.yaml file in your preferred text editor and examine its content for the service account definition to be used for this tutorial.

```bash
kubectl apply -f k8s/vault-auth-service-account.yam
```

### Kubernetes 1.24+ only

The service account generated a secret that is required for configuration automatically in Kubernetes 1.23. In Kubernetes 1.24+, you need to create the secret explicitly. You can use `kubectl get nodes` to get your version

```bash
kubectl apply -f k8s/vault-auth-secret.yam
```

Now we must get the new secret name and token, we will store them as env variables: `SA_SECRET_NAME` `SA_JWT_TOKEN` `SA_CA_CRT` `K8S_HOST`

```bash
export SA_SECRET_NAME=$(kubectl get secrets --output=json \
    | jq -r '.items[].metadata | select(.name|startswith("vault-auth-")).name')
export SA_JWT_TOKEN=$(kubectl get secret $SA_SECRET_NAME \
    --output 'go-template={{ .data.token }}' | base64 --decode)
export SA_CA_CRT=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
export K8S_HOST=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.server}')
```

## Configure Vault
Now connect to Vault, enable and configure Kubernetes authentication, KV secrets engine, a role and policy for Kubernetes, and create a static secret

```bash
# Login To Vault
export VAULT_ADDR='http://127.0.0.1:8200'
vault login

# Create policy
vault policy write myapp-kv-ro - <<EOF
path "secret/data/myapp/*" {
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
     bound_service_account_namespaces=default \
     token_policies=myapp-kv-ro \
     ttl=24h
```

## Testing K8s Auth

Here we are going to test that auth works using a simple app with the image `burtlo/devwebapp-ruby:k8s`, you can skip this step

```bash
export VAULT_ADDR='http://127.0.0.1:8200'

kubectl apply -f auth-test/devwebapp.yaml

kubectl get pods

kubectl exec -ti devwebapp -- /bin/sh
```

Set `KUBE_TOKEN` to the service account token inside the devwebapp and authenticate with Vault through the example role with the KUBE_TOKEN. **Use the external IP address for Vault here**

```bash
export KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
export VAULT_ADDR='http://192.168.0.238:8200'

curl --request POST --data '{"jwt": "'"$KUBE_TOKEN"'", "role": "example"}' $VAULT_ADDR/v1/auth/demo-auth-mount/login
```

Clean up the temp container
```bash
kubectl delete -f auth-test/devwebapp.yaml
```

## Deploy the 