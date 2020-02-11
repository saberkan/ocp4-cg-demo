# Présenation
## Appliquer une configuration automatiquement par namespace en utilisant les labels
## Comme pour les network policies (multitenant), ou les quotas

# Utilisation
## Installation via operatorhub
<pre>
Namespace Configuration Operator
</pre>

## Création d'un configuration stockage small
<pre>
apiVersion: redhatcop.redhat.io/v1alpha1
kind: NamespaceConfig
metadata:
  name: small-size
spec:
  selector:
    matchLabels:
      size: small  
  resources:
  - apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: small-size  
    spec:
      hard:
        requests.cpu: "4"
        requests.memory: "2Gi"
</pre>

## Creation de network policy multitenant