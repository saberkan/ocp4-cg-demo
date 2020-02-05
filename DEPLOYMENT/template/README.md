# Nous pouvons également utiliser des templates d'application avec place holders pour déployer

# Lister les templates dispo par defautl
<pre>
oc get template -n openshift
</pre>

# Une template il prend des parametres 
<pre>
oc get template mongodb-ephemeral -o yaml -n openshift | grep -A 10 -i 'parameters'
</pre>

# pour générer une liste des resources depuis un template existant
<pre>
oc process template/mongodb-ephemeral -n openshift MEMORY_LIMIT=512
</pre>

# Pour crée
<pre>
oc process template/mongodb-ephemeral -n openshift MEMORY_LIMIT=512 | oc create -f -
</pre>