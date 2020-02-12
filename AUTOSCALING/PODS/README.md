# Présentation
## Scalup out pods selon consommation RAM CPU
## RAM (Tech preview)

# Utilisation
## Par ligne de commande
<pre>
oc autoscale dc/image-registry --min 1 --max 10 --cpu-percent=50
</pre>

## Par Yaml
<pre>
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: cache 
spec:
  maxReplicas: 5 
  minReplicas: 1 
  scaleTargetRef:
    apiVersion: apps.openshift.io/v1
    kind: DeploymentConfig 
    name: cache  
  targetCPUUtilizationPercentage: 50 
</pre>

## Reproduire 
<pre>
oc new-app openshift/hello-openshift:v3.10 --name=cache     -n scheduling

# Edit dc/cache resources to set limits
resources:
    limits:
      cpu: 100m
      memory: 20Mi
    requests:
      cpu: 50m
      memory: 15Mi

# Create autoscaller
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: cache 
spec:
  maxReplicas: 5 
  minReplicas: 1 
  scaleTargetRef:
    apiVersion: apps.openshift.io/v1
    kind: DeploymentConfig 
    name: cache  
  targetCPUUtilizationPercentage: 40

 # Voir les events

 # Requete sur le pod
</pre>


# Pour aller plus loin
https://docs.openshift.com/container-platform/4.3/monitoring/exposing-custom-application-metrics-for-autoscaling.html

# Hors sujet culture générale, VM sur kube
https://github.com/kubevirt/kubevirt