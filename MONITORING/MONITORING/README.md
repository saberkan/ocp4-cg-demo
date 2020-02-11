# Create the monitoring stack from operatorhub

# Monitoring
## Create prometheus instance
<pre>
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: prometheus
  namespace: mycustom-monitoring
spec:
  replicas: 1
  resources:
    limits:
      cpu: '2'
      memory: 500Mi
    requests:
      cpu: '1'
      memory: 250Mi
  ruleSelector: {}
  securityContext: {}
  serviceAccountName: prometheus-k8s
  serviceMonitorSelector: {}
  storage:
    volumeClaimTemplate:
      spec:
        resources:
          requests:
            storage: 4Gi
        storageClassName: ocs-storagecluster-ceph-rbd-retain //use the needed storage class
</pre>

## Create service monitor to scrape metrics from my namespace
<pre>
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: prometheus
  name: infra-metrics
  namespace: mycustom-monitoring
spec:
  endpoints:
    - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      interval: 30s
      params:
        'match[]':
          - '{namespace=~"saberkan-.*"}'
      path: /federate
      port: web
      scheme: https
      tlsConfig:
        caFile: /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
        serverName: prometheus-k8s.openshift-monitoring.svc.cluster.local
  namespaceSelector:
    matchNames:
      - openshift-monitoring
  selector:
    matchLabels:
      prometheus: k8s
</pre>

## Add role binding for promehteus sa
<pre>

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: prometheus-k8s-extended-tinubu
subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: mycustom-monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-k8s-extended


apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s-extended
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - get
- nonResourceURLs:
  - '*'
  verbs:
  - get
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
</pre>

## A ce stade vérifier si les metrics remontent