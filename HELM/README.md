/!\ TACH PREVIEW SUR 4.3

source: 
https://blog.openshift.com/openshift-4-3-deploy-applications-with-helm-3/
https://docs.openshift.com/container-platform/4.3/cli_reference/helm-cli/getting-started-with-helm-on-openshift-container-platform.html

# Presentation
- Helm 3 seulement (Tiller is gone)
- Utilisation d'un repo de charts pour déployer des applications
- Approché 3 étapes, lire l'etat des objets, mettre à jour. (sans perte des modifications manuels)

# Installation de la cli
<pre>
source: https://docs.openshift.com/container-platform/4.3/cli_reference/helm-cli/getting-started-with-helm-on-openshift-container-platform.html#installing-helm_getting-started-with-helm-on-openshift
</pre>

# Utilisation
<pre>
oc new-project saberkan-helm
</pre>

# Ajout d'un nouveau repo
<pre>
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
</pre>

# Deployer une application depuis un repo
<pre>
helm install example-mysql stable/mysql
helm list
</pre>

# Upgrade 
<pre>
helm upgrade --install <release name> --values <values file> <chart directory>
</pre>

# Création d'une helm chart pour une application 
## Génération des objets
<pre>
oc project saberkan-helm
oc new-app centos/ruby-25-centos7~https://github.com/saberkan/ruby-ex.git
</pre>

## Attendre que l'image soit buildée, et la partager sur le namespace openshift (afin qu'elle soit reutilisable par tout le cluster)
<pre>
oc tag saberkan-helm/ruby-ex:latest openshift/ruby-ex:latest
</pre>

## Editer le deploiement config pour utiliser l'image depuis le namespace openshift
<pre>
oc edit dc
...
  triggers:
  - type: ConfigChange
  - imageChangeParams:
      automatic: true
      containerNames:
      - ruby-ex
      from:
        kind: ImageStreamTag
        name: ruby-ex:latest
        namespace: openshift
...
</pre>

# Expose
<pre>
oc expose svc/ruby-ex
</pre>

## recuper les objets nécessaires pour deploiement de l'application
<pre>
oc get dc/ruby-ex -o yaml > dc.yaml
oc get svc/ruby-ex -o yaml > svc.yaml
oc get route/ruby-ex -o yaml > route.yaml
</pre>

## Editer le deployment config pour le rendre reutilisable
<pre>
vim dc.yaml
metadata:
  labels:
    app: ruby-ex
  name: ruby-ex
spec:
...
# Ensuite supprimer la partie status
</pre>

## Editer le service pour le rendre reutilisable
<pre>
vim svc.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ruby-ex
  name: ruby-ex
spec:
  ports:
  - name: 8080-tcp
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: ruby-ex
    deploymentconfig: ruby-ex
  sessionAffinity: None
  type: ClusterIP
</pre>

## Editer la route pour le rendre reutilisable
<pre>
vim route.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: ruby-ex
  name: ruby-ex
spec:
  port:
    targetPort: 8080-tcp
  to:
    kind: Service
    name: ruby-ex
    weight: 100
  wildcardPolicy: None
</pre>

## Initier le helm chart
<pre>
helm create ruby-ex
</pre>

## Edit ruby chart file
<pre>
vim ruby-ex/Chart.yaml 
apiVersion: v2
name: ruby-ex
description: A Helm chart for Kubernetes to deploy ruby-ex
type: application
version: 0.1.0
appVersion: 1.0.0
</pre>

## Il faut remarquer que des valeurs par default ont été générés
<pre>
Values.yaml : ce sont les variables à utiliser dans les placeholders
Templates/NOTES.txt : c'est le message qui d'affiche à la création de template
Templates/_helpers.tpl : des traitement supplémentaires pour éditer des objets, préparer des services accounts etc
</pre>

## Nous allons templatiser no resources yaml
<pre>
# Deployement config
    app: {{ .Values.nameOverride }}
  name: {{ .Chart.Name }}
  replicas: {{ .Values.replicaCount }}
    app: {{ .Values.nameOverride }}
    resources: {}
        app: {{ .Values.nameOverride }}
        - containerPort: {{ .Values.service.port }}
        resources: {}
      securityContext: {}
        name: {{ .Values.image.name }}:{{ .Values.image.version }}
        namespace: {{ .Values.image.namespace }}
# svc
    app: {{ .Values.nameOverride }}
    port: {{ .Values.service.port }}
    app: {{ .Values.nameOverride }}
  type: {{ .Values.service.type  }}

# route
    app: {{ .Values.nameOverride }}
</pre>

# Editer values.yaml

## Les mettre dans la chart
<pre>
# Nettoyer le reste la charte
rm ruby-ex/templates/{deployment,ingress,serviceaccount,service}.yaml
rm ruby-ex/templates/NOTES.txt
mv dc.yaml ruby-ex/templates/                                                                                  
mv svc.yaml ruby-ex/templates/
mv route.yaml ruby-ex/templates/ 
</pre>

## Deployer
<pre>
oc new-project saberkan-test-helm-1
helm template ruby-saberkan-1 ./ruby-ex --set service.port=8080
helm install ruby-ex ./ruby-ex --set service.port=8080
</pre>

## Check installed
<pre>
helm list
# if failed
helm delete ruby-saberkan-1
helm install ruby-saberkan-1 ./ruby-ex --set service.port=8080
</pre>

## Archive my chart
<pre>
helm package ruby-ex

</pre>

## Use repository
<pre>
https://helm.sh/docs/topics/chart_repository/touche 
# Create git repo helm-charts
mkdir helm-charts && cd helm-charts
echo "# helm-charts" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/saberkan/test-helm-charts.git
git push -u origin master
# Next, you’ll want to make sure your gh-pages branch is set as Github Pages
# check : https://saberkan.github.io/helm-charts/
mkdir saberkan-charts
mv ../ruby-ex-0.1.0.tgz saberkan-charts/
helm repo index saberkan-charts/
helm repo add saberkan-charts https://saberkan.github.io/helm-charts/saberkan-charts
helm search repo ruby
</pre>

## Package
<pre>
helm package ruby-ex/
helm install ruby-saberkan-1 ruby-ex-0.1.0.tgz --set service.port=8080
</pre>

## Use dependencies
