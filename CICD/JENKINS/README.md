# Différents strategies
## FULL OCP : Utilisation des jenkins par projet dans OpenShift avec la stratégie jenkins pipeline
## MID OCP 1 : Instanciation d'un jenkins commun dans openshift, et le configurer pour utiliser le plugin kubernetes
## MID OCP 2 : Utiliser une jenkins externe, le configurer pour utiliser le plugin kubernetes
Remarque: le plugin kubernetes permet d'executer des jobs dans pods dédiés (temporaires)
Sources pour mode MID OCP : 
https://github.com/jenkinsci/kubernetes-plugin

Il exite également une DSL openshift pour intéragir avec openshift
https://blog.openshift.com/using-openshift-pipeline-plugin-external-jenkins/


# Attention à partir de la 4.3 Red Hat recommande plutot d'utiliser Tekton, ou bien d'executer la pipeline en dehors d'ocp sur jenkins
source: https://github.com/openshift/pipelines-tutorial/

## Deployer un jenkins depuis le catalogue developpeur
## Préparation des namespaces 
<pre>
oc new-project development-saberkan
oc new-project staging-saberkan
oc new-project pipeline-saberkan
</pre>

## Ajouter des droits au utilisateurs est service account
<pre>
oc adm policy add-role-to-user admin developer -n development-saberkan
oc adm policy add-role-to-user admin developer -n staging-saberkan
oc adm policy add-role-to-user admin developer -n pipeline-saberkan
oc adm policy add-role-to-user admin developer -n openshift
oc adm policy add-role-to-user admin system:serviceaccount:pipeline-saberkan:jenkins -n development-saberkan
oc adm policy add-role-to-user admin system:serviceaccount:pipeline-saberkan:jenkins -n staging-saberkan
oc adm policy add-role-to-user admin system:serviceaccount:pipeline-saberkan:jenkins -n openshift
oc adm policy add-role-to-user admin system:serviceaccount:development-saberkan:builder -n openshift
</pre>

## Crée un buildconfig de type JenkinsPipeline (/!\ à faire ma manellement)
<pre>
cat <<EOF > bc-pipeline.yaml
kind: "BuildConfig"
apiVersion: "build.openshift.io/v1"
metadata:
  name: "springboot-saberkan-pipeline"
  namespace: "pipeline-saberkan"
spec:
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfile: >-
        node('maven') {
            stage ('Clone resources') {
                git url: 'https://github.com/saberkan/openshift-springboot-pipeline.git', branch: "develop"
            }
            stage ('Test application') {
                sh "cd springboot-project/springboot && mvn clean test"
            }
            stage ('buildInDevelopment') {
                sh 'oc start-build bc/springboot -n development-saberkan'
            }
            stage ('promoteToStaging') {
                def version = input(
                    id: 'userInput', message: 'please provide version to promote to staging',
                    parameters: [[$class: 'StringParameterDefinition', defaultValue: 'None', 
                        description:'version', name:'version']
                    ])
                echo("version: ${version}")

                sh "oc tag openshift/springboot:latest openshift/springboot:${version}"
                istag=sh(returnStdout: true, script: 'oc get istag springboot:1.0.0 -n openshift  --no-headers | awk "{print $2}"").trim()
                sh "oc set image dc/springboot *=${istag} -n staging-saberkan"
                sh "oc rollout latest dc/springboot -n staging-saberkan"
            }
            
        }
    type: JenkinsPipeline
EOF
oc create -f bc-pipeline.yaml -n pipeline-saberkan
</pre>

## Importer image S2I utilisée pour le build de l'application
<pre>
docker pull appuio/s2i-maven-java 
oc import-image s2i-maven-java:latest --from=appuio/s2i-maven-java --confirm -n openshift
</pre>

## Creation des objets du namespace developpement
<pre>
cat <<EOF > dev-objects.yaml
apiVersion: v1
kind: Template
metadata:
  creationTimestamp: null
  name: springboot
objects:
- apiVersion: v1
  kind: Route
  metadata:
    labels:
      app: springboot
    name: springboot
  spec:
    port:
      targetPort: 8080-tcp
    to:
      kind: Service
      name: springboot
      weight: 100
    wildcardPolicy: None
- apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: springboot
    name: springboot
  spec:
    ports:
    - name: 8080-tcp
      port: 8080
      protocol: TCP
      targetPort: 8080
    selector:
      deploymentconfig: springboot
    sessionAffinity: None
    type: ClusterIP
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    annotations:
      openshift.io/generated-by: OpenShiftWebConsole
    creationTimestamp: null
    generation: 3
    labels:
      app: springboot
    name: springboot
  spec:
    replicas: 1
    selector:
      app: springboot
      deploymentconfig: springboot
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
    template:
      metadata:
        annotations:
          openshift.io/generated-by: OpenShiftWebConsole
        creationTimestamp: null
        labels:
          app: springboot
          deploymentconfig: springboot
      spec:
        containers:
        - env:
          - name: application_name
            valueFrom:
              configMapKeyRef:
                key: application_name
                name: springboot-cm
          - name: environment
            valueFrom:
              configMapKeyRef:
                key: environment
                name: springboot-cm
          - name: port
            valueFrom:
              configMapKeyRef:
                key: port
                name: springboot-cm
          - name: version
            valueFrom:
              configMapKeyRef:
                key: version
                name: springboot-cm
          image: 172.30.1.1:5000/openshift/springboot@sha256:b14e38ae16949d18036073ccd129d56cd6365813f5ceeeadb1219b6605ac4e81
          imagePullPolicy: Always
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /actuator/health
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 38
          name: springboot
          ports:
          - containerPort: 8080
            protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /actuator/health
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 15
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
    test: false
    triggers:
    - type: ConfigChange
    - imageChangeParams:
        automatic: true
        containerNames:
        - springboot
        from:
          kind: ImageStreamTag
          name: springboot:latest
          namespace: openshift
        lastTriggeredImage: 172.30.1.1:5000/openshift/springboot@sha256:b14e38ae16949d18036073ccd129d56cd6365813f5ceeeadb1219b6605ac4e81
      type: ImageChange
- apiVersion: v1
  kind: BuildConfig
  metadata:
    creationTimestamp: null
    labels:
      app: springboot
    name: springboot
  spec:
    nodeSelector: null
    output:
      to:
        kind: ImageStreamTag
        namespace: openshift
        name: springboot:latest
    postCommit: {}
    resources: {}
    runPolicy: Serial
    source:
      contextDir: springboot-project/springboot
      git:
        ref: develop
        uri: https://github.com/saberkan/openshift-springboot-pipeline.git
      type: Git
    strategy:
      sourceStrategy:
        from:
          kind: ImageStreamTag
          name: s2i-maven-java:latest
          namespace: openshift
      type: Source
    triggers: []
EOF
oc process -f dev-objects.yaml | oc create -f -  -n development-saberkan
</pre>

# Création objets environnement staging
<pre>
cat <<EOF > staging-objects.yaml
apiVersion: v1
kind: Template
metadata:
  name: springboot
objects:
- apiVersion: v1
  kind: Route
  metadata:
    annotations:
      openshift.io/host.generated: "true"
    creationTimestamp: null
    labels:
      app: springboot
    name: springboot
  spec:
    port:
      targetPort: 8080-tcp
    to:
      kind: Service
      name: springboot
      weight: 100
    wildcardPolicy: None
- apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: springboot
    name: springboot
  spec:
    ports:
    - name: 8080-tcp
      port: 8080
      protocol: TCP
      targetPort: 8080
    selector:
      deploymentconfig: springboot
    sessionAffinity: None
    type: ClusterIP
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: springboot
    name: springboot
  spec:
    replicas: 1
    selector:
      app: springboot
      deploymentconfig: springboot
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
    template:
      metadata:
        annotations:
          openshift.io/generated-by: OpenShiftWebConsole
        creationTimestamp: null
        labels:
          app: springboot
          deploymentconfig: springboot
      spec:
        containers:
        - env:
          - name: application_name
            valueFrom:
              configMapKeyRef:
                key: application_name
                name: springboot-cm
          - name: environment
            valueFrom:
              configMapKeyRef:
                key: environment
                name: springboot-cm
          - name: port
            valueFrom:
              configMapKeyRef:
                key: port
                name: springboot-cm
          - name: version
            valueFrom:
              configMapKeyRef:
                key: version
                name: springboot-cm
          image: openshift/springboot
          imagePullPolicy: Always
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /actuator/health
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 38
          name: springboot
          ports:
          - containerPort: 8080
            protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /actuator/health
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 15
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
    test: false
EOF
oc process -f staging-objects.yaml | oc create -f -  -n staging-saberkan
</pre>

# Creation image stream dans openshift
<pre>
oc create is springboot -n openshift 
</pre>

# Création des configmaps
<pre>
oc create cm springboot-cm --from-file=configMaps/development/ -n development-saberkan
oc create cm springboot-cm --from-file=configMaps/staging/ -n staging-saberkan
</pre>


