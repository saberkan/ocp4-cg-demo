# Construire un docker file en démarrant avec l'agent jmx
<pre>
FROM openjdk:8-jdk-alpine3.7
  
COPY target/gs-rest-service-0.1.0.jar  app.jar
ADD external-libs .

EXPOSE 8080
EXPOSE 8089
ENTRYPOINT ["java","-javaagent:jmx_prometheus_javaagent-0.12.0.jar=8089:jmx_prometheus_javaagent-default.yaml","-jar","app.jar"]
</pre>

# Build
<pre>
kind: BuildConfig
apiVersion: build.openshift.io/v1
metadata:
  name: sb-project
  namespace: saberkan-jmx
  labels:
    name: docker-build
spec:
  nodeSelector: null
  output:
    to:
      kind: ImageStreamTag
      name: 'sb-project:prometheus-jmx'
  resources: {}
  successfulBuildsHistoryLimit: 5
  failedBuildsHistoryLimit: 5
  strategy:
    type: Docker
    dockerStrategy:
      from:
        kind: ImageStreamTag
        namespace: openshift
        name: 'redhat-openjdk18-openshift:1.5'
  postCommit: {}
  source:
    type: Git
    git:
      uri: 'https://github.com/saberkan/sb-project.git'
      ref: prometheus-jmx
  triggers: []
  runPolicy: Serial
</pre>

# Deploy
<pre>
kind: DeploymentConfig
apiVersion: apps.openshift.io/v1
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
  namespace: saberkan-jmx
  labels:
    app: sb-project
spec:
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
  triggers:
    - type: ConfigChange
    - type: ImageChange
      imageChangeParams:
        automatic: true
        containerNames:
          - sb-project
        from:
          kind: ImageStreamTag
          namespace: saberkan-jmx
          name: 'sb-project:prometheus-jmx'
        lastTriggeredImage: >-
          image-registry.openshift-image-registry.svc:5000/saberkan-jmx/sb-project@sha256:8ab6e5046effdce2e4699f5291a926666e0c0dfa3fc3bb16b0f560ac81dccaa5
  replicas: 1
  revisionHistoryLimit: 10
  test: false
  selector:
    app: sb-project
    deploymentconfig: sb-project
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: sb-project
        deploymentconfig: sb-project
      annotations:
        openshift.io/generated-by: OpenShiftNewApp
    spec:
      containers:
        - name: sb-project
          image: >-
            image-registry.openshift-image-registry.svc:5000/saberkan-jmx/sb-project@sha256:8ab6e5046effdce2e4699f5291a926666e0c0dfa3fc3bb16b0f560ac81dccaa5
          ports:
            - containerPort: 8080
              protocol: TCP
            - containerPort: 8089
              protocol: TCP
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext: {}
      schedulerName: default-scheduler
</pre>

# Crée un service pour exposer les jmx
<pre>

kind: Service
apiVersion: v1
metadata:
  name: sb-project-jmx
  namespace: saberkan-jmx
  labels:
    app: sb-project
    prometheus-jmx: 'true'
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
    prometheus.io/path: /metrics
    prometheus.io/port: '8089'
    prometheus.io/scrape: 'true'
spec:
  ports:
    - name: 8089-tcp
      protocol: TCP
      port: 8089
      targetPort: 8089
  selector:
    app: sb-project
    deploymentconfig: sb-project
  type: ClusterIP
  sessionAffinity: None
</pre>

# Crée le service monitor
<pre>
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: saberkan-jmx
  namespace: tinubu-monitoring
spec:
  endpoints:
    - interval: 30s
      port: 8089-tcp
  namespaceSelector:
       any: true

  selector:
    matchLabels:
          prometheus-jmx: 'true'
</pre>
