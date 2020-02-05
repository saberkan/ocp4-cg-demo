# Les resources requestset limits peuvent etre déclarés de deux facons
## soit directement au niveau des pods (dc, bc, etc)
<pre>
  resources:
    requests: 
       cpu: "100m"
       memory: "256Mi"
       ephemeral-storage: "1Gi"
     limits:
	   cpu: "100m" 
       memory: "256Mi" 
       ephemeral-storage: "1Gi" 
</pre>

## Soit par une configuration qui s'applique par defaut à tout les pods, containers au niveau du namespace	(ca se fait moins que la premiere option)
<pre>
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "core-resource-limits" 
spec:
  limits:
    - type: "Pod"
      max:
        cpu: "2" 
        memory: "1Gi" 
      min:
        cpu: "200m" 
        memory: "6Mi" 
    - type: "Container"
      max:
        cpu: "2" 
        memory: "1Gi" 
      min:
        cpu: "100m" 
        memory: "4Mi" 
      default:
        cpu: "300m" 
        memory: "200Mi" 
      defaultRequest:
        cpu: "200m" 
        memory: "100Mi" 
      maxLimitRequestRatio:
        cpu: "10" 
</pre>