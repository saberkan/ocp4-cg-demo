# Présentation
Utilisation des selectors pour placer les pods

# Utilisation
## Selectionner un label ou en ajouter un nouveau pour nos nodes
Ici nous allons utiliser failure-domain.beta.kubernetes.io/zone
<pre>
oc get nodes -L failure-domain.beta.kubernetes.io/zone
</pre>

## Affichage des nodes workers
<pre>
oc get nodes -l node-role.kubernetes.io/worker
</pre>


## Creation des resources
<pre>
oc new-project scheduling
oc new-app openshift/hello-openshift:v3.10 --name=app-zone-a -n scheduling
</pre>

## Vérifier ou tourne les pods
<pre>
oc get pods -o wide
</pre>

## Mettre à jour les deplotment config pour choisir la région
<pre>
spec:
 template:
  spec:
  ...	
      nodeSelector:
        failure-domain.beta.kubernetes.io/zone: us-east-1a
  ...
</pre>

## Vérifier les workrs de cette zone
<pre>
oc get nodes -L failure-domain.beta.kubernetes.io/zone=us-east-1a | grep -i worker
</pre>

## Vérifier ou tourne les pods
<pre>
oc get pods -o wide
</pre>

## Attention sur openshift 4 sur cloud provider, si besoin d'utiliser un label custom, le label doit etre mis au niveau du machineset, et non pas du node correctement (parceque node est une resource qui peut etre recrée).