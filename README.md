# Setup env
- brew install vault consul kubectl kubernetes-helm
- brew cask install virtualbox
- brew cask install minikube
- minikube start

# Deploy Consul and Vault to minikube
- helm install consul stable/consul --set Replicas=1
- helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
- helm install vault incubator/vault --set vault.dev=true --set vault.config.storage.consul.address="consul-0:8500",vault.config.storage.consul.path="vault"

# Deploy any DB
- helm repo add bitnami https://charts.bitnami.com/bitnami
- helm install mysql bitnami/mysql --set root.password=password

# Configure vault
- VAULT_POD=$(kubectl get pods --namespace default -l "app=vault" -o jsonpath="{.items[0].metadata.name}")
- export VAULT_TOKEN=$(kubectl logs $VAULT_POD | grep 'Root Token' | cut -d' ' -f3)
- export VAULT_ADDR=http://127.0.0.1:8200
- kubectl port-forward $VAULT_POD 8200 (In separate terminal)
- echo $VAULT_TOKEN | vault login - (Not required in dev mode)
- vault status

# Enable database secrets backend
- vault secrets enable database

# Config a new database connection
- vault write database/config/mysql-database \
     plugin_name=mysql-database-plugin \
     connection_url="{{username}}:{{password}}@tcp(mysql-slave.default.svc.cluster.local:3306)/" \
     allowed_roles="mysql-role" \
     username="root" \
     password="password"

# Create a vault role to connect to DB and create New Temporary User
- vault write database/roles/mysql-role \
     db_name=mysql-database \
     creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
     default_ttl="1h" \
     max_ttl="24h"

# Create a vault policy that can issue commands to read, create and update DB credentials
- vault policy write mysql-policy mysql-policy.hcl

# Create a K8s service account to authenticate with review token
- kubectl apply -f mysql-serviceaccount.yml

# Extract k8s JWT, CA certificate and the host to configure vault's k8s auth backend
- export VAULT_SA_NAME=$(kubectl get sa mysql-vault -o jsonpath="{.secrets[*]['name']}")
- export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
- export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
- export K8S_HOST=$(kubectl exec consul-0 -- sh -c 'echo $KUBERNETES_SERVICE_HOST')

# Enable K8s auth backend on vault
- vault auth enable kubernetes

# Configure K8s backend auth
- vault write auth/kubernetes/config \
    token_reviewer_jwt="$SA_JWT_TOKEN" \
    kubernetes_host="https://$K8S_HOST:443" \
    kubernetes_ca_cert="$SA_CA_CRT"
- vault write auth/kubernetes/role/mysql \
    bound_service_account_names=mysql-vault \
    bound_service_account_namespaces=default \
    policies=mysql-policy \
    ttl=24h

# Get vault & mysql service name
- kubectl get svc -l app=vault -o jsonpath="{.items[0].metadata.name}"
- kubectl get svc -l app=mysql -o jsonpath="{.items[0].metadata.name}"

# Create a temporary pod for testing
- kubectl run tmp --rm -i --tty --serviceaccount=mysql-vault --image alpine

# Create user and check connection
- apk update
- apk add curl mysql-client jq
- KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
- VAULT_K8S_LOGIN=$(curl --request POST --data '{"jwt": "'"$KUBE_TOKEN"'", "role": "mysql"}' http://vault:8200/v1/auth/kubernetes/login)
- X_VAULT_TOKEN=$(echo $VAULT_K8S_LOGIN | jq -r '.auth.client_token')
- MYSQL_CREDS=$(curl --header "X-Vault-Token:$X_VAULT_TOKEN" http://vault:8200/v1/database/creds/mysql-role)
- echo $MYSQL_CREDS | jq #(This will give the credentials)