#############################
# Installation
#############################
# Installation de Vault
oc new-project hashicorp-vault

# Donner les droits admin au service account
oc adm policy add-scc-to-user privileged -z default

# Configuration vault en configmap
<pre>
cat <<EOF > vault-config.json
{
  "backend": {
    "file": {
      "path": "/vault/file"
    }
  },
  "default_lease_ttl": "168h",
  "max_lease_ttl": "720h",
  "disable_mlock": true,
  "ui": true,
  "listener": {
    "tcp": {
      "address": "0.0.0.0:8200",
      "tls_cert_file": "/var/run/secrets/kubernetes.io/certs/tls.crt",
      "tls_key_file": "/var/run/secrets/kubernetes.io/certs/tls.key"
    }
  }
}
EOF
</pre>
oc create configmap vault-config --from-file=vault-config=./vault-config.json

# Création Vault instance
<pre>
cat <<EOF > vault.yaml
apiVersion: v1
kind: Service
metadata:
  name: vault
  annotations:
    service.alpha.openshift.io/serving-cert-secret-name: vault-cert
  labels:
    app: vault
spec:
  ports:
  - name: vault
    port: 8200
  selector:
    app: vault
---
apiVersion: v1
kind: DeploymentConfig
metadata:
  labels:
    app: vault
  name: vault
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: vault
    spec:
      containers:
      - image: vault:0.11.3
        name: vault
        ports:
        - containerPort: 8200
          name: vaultport
          protocol: TCP
        args:
        - server
        - -log-level=debug    
        env:
        - name: SKIP_SETCAP
          value: 'true' 
        - name: VAULT_LOCAL_CONFIG
          valueFrom:
            configMapKeyRef:
              name: vault-config
              key: vault-config
        volumeMounts:      
        - name: vault-file-backend
          mountPath: /vault/file
          readOnly: false
        - name: vault-cert
          mountPath: /var/run/secrets/kubernetes.io/certs
        livenessProbe:
          httpGet:
            path: 'v1/sys/health?standbyok=true&standbycode=200&sealedcode=200&uninitcode=200'
            port: 8200
            scheme: HTTPS
        readinessProbe:
          httpGet:
            path: 'v1/sys/health?standbyok=true&standbycode=200&sealedcode=200&uninitcode=200'
            port: 8200
            scheme: HTTPS
        securityContext:
          privileged: true
      volumes:
      - name: vault-file-backend
        persistentVolumeClaim:
          claimName: vault-file-backend
      - name: vault-cert
        secret:
          secretName: vault-cert          
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: vault-file-backend
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
</pre>
oc create -f vault.yaml

# Expose route vault
oc create route reencrypt vault --port=8200 --service=vault

# Get route

# Créer root token vault
vault operator init -tls-skip-verify -key-shares=1 -key-threshold=1
Unseal Key 1: /freZZBd6yDqMVLpT9qBwQ7pknOv0E9KjgTe83yjfX4=
Initial Root Token: H6YoC2sPBZc16V5KG2VOd0O6

export KEYS=/freZZBd6yDqMVLpT9qBwQ7pknOv0E9KjgTe83yjfX4=
export ROOT_TOKEN=H6YoC2sPBZc16V5KG2VOd0O6
export VAULT_TOKEN=$ROOT_TOKEN


# Create service account pour communiqer avec kube
oc create sa vault-auth
oc adm policy add-cluster-role-to-user system:auth-delegator system:serviceaccount:hashicorp-vault:vault-auth

# Recupération token service account, et CA vault
reviewer_service_account_jwt=$(oc serviceaccounts get-token vault-auth)
pod=$(oc get pods -lapp=vault --no-headers -o custom-columns=NAME:.metadata.name)
oc exec $pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt > /tmp/ca.crt

# Activer le plugin kubernetes
vault auth enable -tls-skip-verify kubernetes

# Ajouter le service account vault-auth sur vault pour communiquer avec kube
export OPENSHIFT_HOST=https://api.cluster-mpl-d015.mpl-d015.example.opentlc.com:6443
vault write -tls-skip-verify auth/kubernetes/config token_reviewer_jwt=$reviewer_service_account_jwt kubernetes_host=$OPENSHIFT_HOST kubernetes_ca_cert=@/tmp/ca.crt

#############################
# DECLARATION DES ROUSOURCES
#############################
# Ecrire une policy pour un secret
cat <<EOF > policy-monsecret.hcl
path "secret/monsecret" {
  capabilities = ["read", "list"]
}
EOF
vault policy write -tls-skip-verify policy-monsecret policy-monsecret.hcl

# Ecrire mon secret
vault write -tls-skip-verify secret/monsecret password=SECURE

# Donner acces à monsecret par les pods du namespace monnamespace pendant 2h
vault write -tls-skip-verify auth/kubernetes/role/monrole-monnamespace-monsecret \
    bound_service_account_names=default bound_service_account_namespaces='monnamespace' \
    policies=policy-monsecret \
    ttl=2h

# Lire mon secret  en tant que root
vault read -tls-skip-verify secret/monsecret

#############################
# Utilisation
#############################
# Lire mon secret en tant que user du namespace monnamespace, Création namespace, recupération token default, connexion
oc new-project monnamespace
default_account_token=$(oc sa get-token default -n monnamespace)
vault write -tls-skip-verify auth/kubernetes/login role=monrole-monnamespace-monsecret jwt=${default_account_token}
export VAULT_TOKEN=w39Pt4QG5NljuduCwZFYDIEU
vault read -tls-skip-verify secret/monsecret


#############################
# Utilisation avec openshift
#############################
# Deployer une application exemple
oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git

# Editer le deployment config pour charger les secrets au démarrage avec un init container
<pre>
    spec:
      volumes:
      - name: app-secrets
        emptyDir: {}
      - name: vault-token
        emptyDir: {}
      initContainers:
      - name: vault-init
        image: quay.io/lbroudoux/ubi8:latest
        command:
          - "sh"
          - "-c"
          - >
            OCP_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token);
            curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "monrole-monnamespace-monsecret"}' https://vault-hashicorp-vault.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com/v1/auth/kubernetes/login | jq -j '.auth.client_token' > /etc/vault/token;
            X_VAULT_TOKEN=$(cat /etc/vault/token);
            curl -k --header "X-Vault-Token: $X_VAULT_TOKEN" https://vault-hashicorp-vault.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com/v1/secret/monsecret > /etc/app/monsecret;
            # exemple utiliser jq pour recuperer le secret dans un fichier
            # echo "spring.data.mongodb.uri=mongodb://$(jq -j '.data.user' /etc/app/creds.json):$(jq -j '.data.password' /etc/app/creds.json)@mongodb/sampledb" > /etc/app/application.properties;
        volumeMounts:
        - name: app-secrets
          mountPath: /etc/app
        - name: vault-token
          mountPath: /etc/vault
</pre>
<pre>
# Ajouter le volume partagé sur le main container 
        volumeMounts:
        - name: app-secrets
          mountPath: /etc/app
</pre>

