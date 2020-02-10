# Présentation
## Utilisation des labels pour définir le placement des pods par rapport à d'autres pods
## Use case rapprocher des pods sur le meme host, zone etc. ou plutot eloigner

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

## Deployer 2 applis
<pre>
oc new-app openshift/hello-openshift:v3.10 --name=cache     -n scheduling
oc new-app openshift/hello-openshift:v3.10 --name=webserver -n scheduling
</pre>

## Nous allons utiliser le label app=cache pour dispatcher les pods cache sur tous les serveurs
<pre>
# Editer DC sur la console 
...
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: "kubernetes.io/hostname"
...

oc scale dc cache --replicas=4
# remarque quand y'a 3 nodes, le 4eme ne démarre pas, ne peut tourner sur le meme node selon la regle
oc scale dc cache --replicas=2
oc delete pods -l app=cache
oc delete pods cache-2-deploy
</pre>

## Maintenant nous allons setter web server pour anti affinié par rapport au node le concernant, et affinité par rapport au cache
<pre>
...
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - webserver
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: "kubernetes.io/hostname"
...

oc get pods -o wide | grep -i running
oc delete pods -l app=webserver	
</pre>