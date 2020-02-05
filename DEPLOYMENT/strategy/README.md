# On va partir d'une image existante pour générer un deploiement config
# Nous avons selectionner image-registry.openshift-image-registry.svc:5000/openshift/httpd


# Génération du déploiement config, svc, is
<pre>
oc new-app --docker-image=image-registry.openshift-image-registry.svc:5000/openshift/httpd --insecure-registry
</pre>

# Exposition de l'application
<pre>
oc expose svc httpd	
oc get route
httpd-ab-testing.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com
</pre>

# Vérifier l'accés à l'application
<pre>
curl -X GET httpd-ab-testing.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com
</pre>

# Par default l'application a la staratgy rollout
<pre>
  strategy:
    type: Rolling
    rollingParams:
      updatePeriodSeconds: 1
      intervalSeconds: 1
      timeoutSeconds: 600
      maxUnavailable: 25%
      maxSurge: 25%
    resources: {}
    activeDeadlineSeconds: 21600
</pre>
Ceci signifie que dans une durée de 21600, Le rodeployement va etre tenté avec les options suivante.
maxSurge : nous pouvons dépasser de 25% le nombre de pods demandés durant le rolling
maxUnavailable : 25% de pods doivent forcement resté up

# On scale up le deploiement à 4, puis on le relance pour étudier le comportement
<pre>
oc scale dc/httpd --replicas=4
oc get pods | grep http
httpd-1-4svzz      1/1     Running     0          15s
httpd-1-btmm8      1/1     Running     0          11m
httpd-1-g45ng      1/1     Running     0          15s
httpd-1-tlt5d      1/1     Running     0          15s

oc rollout latest dc/httpd
oc get pods | grep httpd
httpd-1-4svzz      0/1     Terminating         0          104s
httpd-1-btmm8      1/1     Running             0          13m
httpd-1-g45ng      0/1     Terminating         0          104s
httpd-1-tlt5d      1/1     Terminating         0          104s
httpd-2-54b8j      0/1     ContainerCreating   0          1s
httpd-2-6b4pp      1/1     Running             0          11s
httpd-2-6dp2g      1/1     Running             0          14s
httpd-2-rgqtq      0/1     ContainerCreating   0          4s
</pre>

# Voir également sur l'interface developpeur le comportement

# Ensuite nous testons la stratégie recreate
<pre> 
oc edit dc/httpd
# Remplacer 
  strategy:
    activeDeadlineSeconds: 21600
    resources: {}
    rollingParams:
      intervalSeconds: 1
      maxSurge: 25%
      maxUnavailable: 25%
      timeoutSeconds: 600
      updatePeriodSeconds: 1
    type: Rolling
# Par
  strategy:
    type: Recreate
</pre>

# Relancer le déploiement et vérifier le comportement
<pre>
oc rollout latest dc/httpd
oc get pods | grep httpd
httpd-3-672bj      0/1     Terminating   0          16m
httpd-3-7jwr5      0/1     Terminating   0          16m
httpd-3-deploy     0/1     Completed     0          16m
httpd-4-deploy     1/1     Running       0          14s
oc get pods | grep httpd

httpd-4-8hdf6      0/1     ContainerCreating   0          6s
httpd-4-deploy     0/1     Completed           0          31s
httpd-4-hg4lw      0/1     ContainerCreating   0          6s
httpd-4-rpcxf      1/1     Running             0          15s
httpd-4-vsmzf      0/1     ContainerCreating   0          6s
</pre>

# Pour information, il est possible de faire des traitement avant (lifecyclehook), durant et apres deploiement de l'application . Or, nous préférons utiliser des inticontainers, et sidecard pour cette finalité (voir lab VAULT)
Nous l'exemple suivant, un script run.sh est lancé dans un container elasticsreach avant début de déploiement
<pre>
  recreateParams: 
    pre:
      failurePolicy: Abort
      execNewPod:
        command: [ "./run.sh" ]
        containerName: "elasticsearch"
    mid: {}
    post: {}
</pre>

# Il est également possible de mettre en place des stréatégies custom, en revanche ceci est très peu utilisé
<pre>
spec:
  replicas: 1
  strategy:
    resources: {}
    type: Custom
    customParams:
      image: openshift/origin-deployer
      command:
      - /bin/sh
      - -c
      - >
        # Pre-deployment code goes here

        oc scale replicationcontrollers/${OPENSHIFT_DEPLOYMENT_NAME}
        --replicas=1
        -n ${OPENSHIFT_DEPLOYMENT_NAMESPACE}
        --token=${LOGIN_TOKEN}
        --server=${MASTER_SERVER}
        --insecure-skip-tls-verify;

        # Post-deployment code goes here.

        # TODO: Figure out how to make sure the deployment is complete 
        #       and the pod is running before executing any post-deployment code
      environment:
      - name: LOGIN_TOKEN
        value: (* see below)
      - name: MASTER_SERVER
        # Minishift login URL
        value: https://192.168.42.229:8443
</pre>