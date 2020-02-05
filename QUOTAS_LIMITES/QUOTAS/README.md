# Source : https://docs.openshift.com/container-platform/4.3/applications/quotas/quotas-setting-per-project.html  
# Il est possible de définir 3 types de quotas : 
## Des quotas pour le nombre d'objects
## Des quotas pour cpu ram
## Des quotas pour stockage

<pre>
apiVersion: v1
kind: ResourceQuota
metadata:
  name: core-object-counts
spec:
  hard:
    configmaps: "10" 
    persistentvolumeclaims: "4" 
    replicationcontrollers: "20" 
    secrets: "10" 
    services: "10" 
    services.loadbalancers: "2" 
</pre>

<pre>
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "4" 
    requests.cpu: "1" 
    requests.memory: 1Gi 
    requests.ephemeral-storage: 2Gi 
    limits.cpu: "2" 
    limits.memory: 2Gi 
    limits.ephemeral-storage: 4Gi 
</pre>


# Les quotas de resoucrces et storage s'appliquent au pods par rapport à leurs scopes
<pre>
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort
spec:
  hard:
    pods: "1" 
  scopes:
  - BestEffort 
</pre>
## Terminating : Match pods where spec.activeDeadlineSeconds >= 0.
## Il s'agit des pods qui ont un parametre spec.activeDeadlineSeconds >= 0, ce sont les pods utilisés par les applications comme pour un deploymentconfig classique.

## NotTerminating : Match pods where spec.activeDeadlineSeconds is nil.
Il s'agit des pods sans parametre spec.activeDeadlineSeconds, comme par exemple les batch (jobs) ou buildconfigs

## BestEffort : Match pods that have best effort quality of service for either cpu or memory.
Il s'agit par defaut des pods qui ont une request == limits
<pre>
resources:
  limits:
    memory: "200Mi"
    cpu: "700m"
  requests:
    memory: "200Mi"
    cpu: "700m"
</pre>

<pre>
spec:
  containers:
    ...
    resources: {}
  ...
  qosClass: BestEffort
</pre>

## NotBestEffort : Match pods that do not have best effort quality of service for cpu and memory.

# /!\ Attention une fois les quotas sont configurer au niveau des namespace, il est indisponsable de déclaer des resources de consommation. Sinon les pods risquent de ne pas démarer. voir section (LIMITS)


