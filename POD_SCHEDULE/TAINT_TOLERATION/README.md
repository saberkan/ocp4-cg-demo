# Présentation
## Configurer les nodes pour recevoir un type spécifique de pods


# Deployer applications
<pre>
oc new-app openshift/hello-openshift:v3.10 --name=cache     -n scheduling
oc new-app openshift/hello-openshift:v3.10 --name=webserver -n scheduling
oc scale dc/cache --replicas=2
oc scale dc/webserver --replicas=2
oc get pod -o wide|grep Running
</pre>

## Configurer un node pour recevoir que les pods cache
<pre>
oc adm taint nodes ip-10-0-137-137.ec2.internal montag=true:NoSchedule
</pre>

## Sur le cloud provider, le taint doit etre posé au niveau du machineset, et non pas au niveau du node.
<pre>
spec:
  template:
    spec:
      taints:
      - key: montag
        value: "true"
        effect: NoSchedule
</pre>

## Mettre la tolération au niveau de l'application pour deployer tout au niveau du node
<pre>
dans le dc
...
tolerations:
 - key: "montag"
   operator: "Equal"
   value: "true"
   effect: "NoSchedule"
...

# Nous pouvons constater que seul l'application cache a le droit d'aller sur ce node (ip-10-0-137-137.ec2.internal)
oc get pods -o wide | grep -i running
</pre>

## Le meme principe est utilisé pour les masters