# Présentation
## Régulation du traffic à l'interieur du cluster
## 2 modes possibles: multitenant, network policy (OpenShift SDN Container Network Interface)
## Par default mode Network policy

# Joindre le reseau des projects en cas de multitenant
## Joindre les projects
<pre>
oc adm pod-network join-projects --to=<project1> <project2> <project3>
</pre>

## Ce qui se passe au niveau d'ocp
<pre>
oc get netnamespace
</pre>

# En mode network policy
## Tous les pods du cluster peuvent communiquer

## Regulation du traffic avec les network policy, autoriser le router
<pre>
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress
  podSelector: {}
  policyTypes:
  - Ingress
</pre>

## Autoriser le meme namespace
<pre>
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-same-namespace
spec:
  podSelector:
  ingress:
  - from:
    - podSelector: {}
</pre>

## Autoriser la communication entre 2 pods
<pre>
kind: NetworkPolicy
apiVersion: extensions/v1beta1
metadata:
  name: allow-27107 
spec:
  podSelector: 
    matchLabels:
      app: mongodb
  ingress:
  - from:
    - podSelector: 
        matchLabels:
          app: app
    ports: 
    - protocol: TCP
      port: 27017
</pre>

## Bloquer le reste
<pre>
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-by-default
spec:
  podSelector:
  ingress: []
</pre>

