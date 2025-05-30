# Vault-K8s

## Clone github repo to setup kind cluster
```
git clone https://github.com/shabbirsaifee92/multi-node-kind-cluster.git
cd multi-node-kind-cluster
```

## Create kubernetes cluster
```
./create_cluster.sh
```

## Add hashicorp helm repository
```
 helm repo add hashicorp https://helm.releases.hashicorp.com
```

## Deploy hashicorp vault
```
helm install vault hashicorp/vault -n vault --create-namespace
```

## k9s
```
brew install k9s
k9s
```

## Vault Intialize and Useal

```
# exec into the pod first

## initialize vault to get root token and unseal keys
vault operator init

## use 3 keys to unseal vault, run this command 3 times
vault operator unseal

## login to vault using root token
vault login <root-token>
```

## Enable kv-v2 secrets engine, add secret and add policy

```
vault secrets list

vault secrets enable kv-v2

vault kv put kv-v2/vault-demo/mysecret foo=bar

vault kv get kv-v2/vault-demo/mysecret

vault policy list

vault policy write mysecret - << EOF
path "kv-v2/vault-demo/mysecret" {
  capabilities = ["read"]
}
EOF
```

## Configure Kubernetes backend, and Role
```
## exec into vault server

vault auth enable kubernetes

vault write auth/kubernetes/config \
    kubernetes_host=https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT
    
vault write auth/kubernetes/role/vault-demo \
    bound_service_account_names=default \
    bound_service_account_namespaces=default \
    policies=default,mysecret \
    ttl=1h
```

## Pod definition

```
# vault-demo.yaml

apiVersion: v1
kind: Pod
metadata:
  name: vault-demo
spec:
  restartPolicy: "OnFailure"
  containers:
    - name: vault-demo
      image: badouralix/curl-jq
      command: ["sh", "-c"]
      resources: {}
      args:
      - |
        VAULT_ADDR="http://vault-internal.vault:8200"
        SA_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
        VAULT_RESPONSE=$(curl -X POST -H "X-Vault-Request: true" -d '{"jwt": "'"$SA_TOKEN"'", "role": "vault-demo"}' \
          $VAULT_ADDR/v1/auth/kubernetes/login | jq .)

        echo $VAULT_RESPONSE
        echo ""

        VAULT_TOKEN=$(echo $VAULT_RESPONSE | jq -r '.auth.client_token')
        echo $VAULT_TOKEN

        echo "Fetching vault-demo/mysecret from vault...."
        VAULT_SECRET=$(curl -s -H "X-Vault-Token: $VAULT_TOKEN" $VAULT_ADDR/v1/kv-v2/data/vault-demo/mysecret)
        echo $VAULT_SECRET

        exit 0;
```
