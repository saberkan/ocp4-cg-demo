#############################
# Installation
#############################
# Installation de Vault
<pre>
oc new-project hashicorp-vault
</pre>

# Donner les droits admin au service account
<pre>
oc adm policy add-scc-to-user privileged -z default
</pre>

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
oc create configmap vault-config --from-file=vault-config=./vault-config.json
</pre>

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
oc create -f vault.yaml
</pre>

# Expose route vault
<pre>
oc create route reencrypt vault --port=8200 --service=vault
</pre>

# Get route
<pre>
oc get route
export VAULT_ADDR=https://vault-hashicorp-vault-2.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com
</pre>

# Créer root token vault
<pre>
vault operator init -tls-skip-verify -key-shares=1 -key-threshold=1
Unseal Key 1: O36nl8nY9A5jNqx1XsTni2VxV9kyn/vq5z4IUNgDxh8=

Initial Root Token: 5YDgYBxfWsN3VaHZ0G464ure


export KEYS=O36nl8nY9A5jNqx1XsTni2VxV9kyn/vq5z4IUNgDxh8=
export ROOT_TOKEN=5YDgYBxfWsN3VaHZ0G464ure
export VAULT_TOKEN=$ROOT_TOKEN

vault operator unseal -tls-skip-verify $KEYS
</pre>


# Create service account pour communiqer avec kube
<pre>
oc create sa vault-auth
oc adm policy add-cluster-role-to-user system:auth-delegator system:serviceaccount:hashicorp-vault-2:vault-auth
</pre>

# Recupération token service account, et CA vault
<pre>
reviewer_service_account_jwt=$(oc serviceaccounts get-token vault-auth)
pod=$(oc get pods -lapp=vault --no-headers -o custom-columns=NAME:.metadata.name)
oc exec $pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt > /tmp/ca.crt
</pre>

# Activer le plugin kubernetes
<pre>
vault auth enable -tls-skip-verify kubernetes
</pre>

# Ajouter le service account vault-auth sur vault pour communiquer avec kube
<pre>
export OPENSHIFT_HOST=https://api.cluster-mpl-d015.mpl-d015.example.opentlc.com:6443
vault write -tls-skip-verify auth/kubernetes/config token_reviewer_jwt=$reviewer_service_account_jwt kubernetes_host=$OPENSHIFT_HOST kubernetes_ca_cert=@/tmp/ca.crt
</pre>

#############################
# DECLARATION DES ROUSOURCES
#############################
# Ecrire une policy pour un secret
<pre>
cat <<EOF > policy-monsecret.hcl
path "secret/monsecret" {
  capabilities = ["read", "list"]
}
EOF
vault policy write -tls-skip-verify policy-monsecret policy-monsecret.hcl
</pre>

# Ecrire mon secret
<pre>
vault write -tls-skip-verify secret/monsecret password=SECURE
</pre>

# Donner acces à monsecret par les pods du namespace monnamespace pendant 2h
<pre>
vault write -tls-skip-verify auth/kubernetes/role/monrole-monnamespace-monsecret \
    bound_service_account_names=default bound_service_account_namespaces='cg-demo-0402' \
    policies=policy-monsecret \
    ttl=2h
</pre>

# Lire mon secret  en tant que root
<pre>
vault read -tls-skip-verify secret/monsecret
</pre>

#############################
# Utilisation
#############################
# Lire mon secret en tant que user du namespace monnamespace, Création namespace, recupération token default, connexion
<pre>
oc new-project monnamespace
default_account_token=$(oc sa get-token default -n cg-demo-0402)
vault write -tls-skip-verify auth/kubernetes/login role=monrole-monnamespace-monsecret jwt=${default_account_token}
export VAULT_TOKEN=w39Pt4QG5NljuduCwZFYDIEU
vault read -tls-skip-verify secret/monsecret
</pre>

#############################
# Utilisation avec openshift
#############################
# Deployer une application exemple
<pre>
oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git
</pre>

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

