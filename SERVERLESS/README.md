Source:
https://access.redhat.com/documentation/en-us/openshift_container_platform/4.2/html/serverless_applications/


# Installation
## Installer l'operateur depuis l'interface

## Crée kubernetes serving sur un namespace
<pre>
apiVersion: v1
kind: Namespace
metadata:
 name: knative-serving
---
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
 name: knative-serving
 namespace: knative-serving

 # Check install
 oc get knativeserving.operator.knative.dev/knative-serving -n knative-serving --template='{{range .status.conditions}}{{printf "%s=%s\n" .type .status}}{{end}}'
</pre>

# Utilisation
## Crée un serverless service
<pre>
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: helloworld-go
  namespace: default
spec:
  template:
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          env:
            - name: TARGET
              value: "Go Sample v1"
</pre>

## Vérifier la route
<pre>
oc get svc -n default
</pre>

# Cli
## Installation
<pre>
Aller en haut à gauche de la console "?"
</pre>

## Lister les services existants
<pre>
kn service list -n default
</pre>

## Création de nouveau service
<pre>
kn service create hello --image gcr.io/knative-samples/helloworld-go --env TARGET=Knative
</pre>

## Mise à jour
<pre>
kn service update hello --env TARGET=Kn -n default
</pre>

## Traffic splitting
<pre>
#Tagger une revision
kn service list -n default
# Prendre le tag dans latest: hello-rrhyr-2
kn service update hello --tag hello-rrhyr-2=current -n default

# Vérifier toute les revisions
kn revision list -n default

# Tagger l'ancienne revision
kn service update hello --tag hello-xprlv-1=old -n default

#Split traffic
kn service update hello --traffic current=50 --traffic old=50 -n default
</pre>