# Présentation
## Scaler up un type de nodes (ex: par zone) quand le nodes est surchargé
## Scaler up un type de nodes (ex: par zone) quand le nodes est surchargé

# Raisons pour que le node soit not ready

# Configuration de l'autoscalling 
<pre>
Machine autoscalling > MachineSet > Machine > node
</pre>

# Exemple:
<pre>
apiVersion: "autoscaling.openshift.io/v1alpha1"
kind: "MachineAutoscaler"
metadata:
  generateName: autoscale-<aws-region-az>-
  namespace: "openshift-machine-api"
spec:
  minReplicas: 1
  maxReplicas: 4
  scaleTargetRef:
    apiVersion: cluster.k8s.io/v1alpha1
    kind: MachineSet
    name: <clusterid>-worker-<aws-region-az>
</pre>