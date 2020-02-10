# Pour aller plus loin
Voir 
## Disruption Budget
<pre>
apiVersion: policy/v1beta1 
kind: PodDisruptionBudget
metadata:
  name: my-pdb
spec:
  selector:  
    matchLabels:
      foo: bar
  minAvailable: 2  
</pre>

## Priority class
<pre>
apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false 
description: "This priority class should be used for XYZ service pods only."

apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
</pre>