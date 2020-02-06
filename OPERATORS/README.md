# Prepare namespace
<pre>
# login as cluster admin first
oc new-project operators
oc create user developer
oc policy add-role-to-user admin -z developer -n operators
oc create user global-admin
oc adm policy add-cluster-role-to-user cluster-admin global-admin
oc import-image rhscl/httpd-24-rhel7 --from=registry.access.redhat.com/rhscl/httpd-24-rhel7 --confirm -n openshift
</pre>

# Create ansible playbook to deploy httpd
## Vérifier openshift list of objects works
<pre>
cat <<EOF > httpd_resources.yaml
apiVersion: v1
items:
- apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: httpd-24-rhel7
    name: httpd-24-rhel7
  spec:
    replicas: 1
    selector:
      app: httpd-24-rhel7
      deploymentconfig: httpd-24-rhel7
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
          app: httpd-24-rhel7
          deploymentconfig: httpd-24-rhel7
      spec:
        containers:
        - env:
          - name: MY_VAR_NAME
            value: IT WORKS
          image: registry.access.redhat.com/rhscl/httpd-24-rhel7@sha256:e4bbea24985cfa97fb7bea65d83ac3a80797a0f2b7f1ffce0b4dd484d2b3458d
          imagePullPolicy: Always
          name: httpd-24-rhel7
          ports:
          - containerPort: 8080
            protocol: TCP
          - containerPort: 8443
            protocol: TCP
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
- apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: httpd-24-rhel7
    name: httpd-24-rhel7
  spec:
    clusterIP: 172.30.161.56
    ports:
    - name: 8080-tcp
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: 8443-tcp
      port: 8443
      protocol: TCP
      targetPort: 8443
    selector:
      deploymentconfig: httpd-24-rhel7
    sessionAffinity: None
    type: ClusterIP
- apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    labels:
      app: httpd-24-rhel7
    name: httpd-24-rhel7
  spec:
    port:
      targetPort: 8080-tcp
    to:
      kind: Service
      name: httpd-24-rhel7
      weight: 100
    wildcardPolicy: None
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
EOF
oc create -f httpd_resources.yaml
</pre>

## Check httpd server is running
## Clean all for now
<pre>
oc delete all -l app=httpd-24-rhel7
</pre>

## Building the playbook
<pre>
mkdir roles && cd roles
ansible-galaxy init --force --offline httpd_openshift
cd ..
</pre>

## Create playbook
<pre>
cat <<EOF > playbook.yaml
---
# - my_var_value : should ve myVarValue on the CR
# - meta.namespace from the CR thanks to ansible-operator

- name: deploy the application
  hosts: localhost
  gather_facts: no
  tasks:
  - name: Set up httpd
    include_role:
      name: ./roles/httpd
    vars:
      _httpd_namespace: "{{ meta.namespace }}"
      _my_var_value: "{{ my_var_value }}"
EOF
</pre>

## Develop role
<pre>
cat <<EOF > roles/httpd_openshift/tasks/main.yml
---
- name: Create deploymentconfig
  k8s:
    state: present
    definition:
        apiVersion: apps.openshift.io/v1
        kind: DeploymentConfig
        metadata:
          namespace: "{{ _httpd_namespace }}"
          labels:
            app: httpd-24-rhel7
          name: httpd-24-rhel7
        spec:
          replicas: 1
          selector:
            app: httpd-24-rhel7
            deploymentconfig: httpd-24-rhel7
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
                app: httpd-24-rhel7
                deploymentconfig: httpd-24-rhel7
            spec:
              containers:
              - env:
                - name: MY_VAR_NAME
                  value: "{{ _my_var_value }}"
                image: registry.access.redhat.com/rhscl/httpd-24-rhel7@sha256:e4bbea24985cfa97fb7bea65d83ac3a80797a0f2b7f1ffce0b4dd484d2b3458d
                imagePullPolicy: Always
                name: httpd-24-rhel7
                ports:
                - containerPort: 8080
                  protocol: TCP
                - containerPort: 8443
                  protocol: TCP
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

- name: Create Service
  k8s:
    state: present
    definition:
        apiVersion: v1
        kind: Service
        metadata:
          namespace: "{{ _httpd_namespace }}"
          labels:
            app: httpd-24-rhel7
          name: httpd-24-rhel7
        spec:
          ports:
          - name: 8080-tcp
            port: 8080
            protocol: TCP
            targetPort: 8080
          - name: 8443-tcp
            port: 8443
            protocol: TCP
            targetPort: 8443
          selector:
            deploymentconfig: httpd-24-rhel7
          sessionAffinity: None
          type: ClusterIP

- name: Create Route
  k8s:
    state: present
    definition:
        apiVersion: route.openshift.io/v1
        kind: Route
        metadata:
          namespace: "{{ _httpd_namespace }}"
          labels:
            app: httpd-24-rhel7
          name: httpd-24-rhel7
        spec:
          port:
            targetPort: 8080-tcp
          to:
            kind: Service
            name: httpd-24-rhel7
            weight: 100
          wildcardPolicy: None
EOF
</pre>


## Check role works
<pre>
ansible-playbook playbook.yaml --extra-vars '{"meta":{"namespace":"operators"},"my_var_value":"PLAYBOOK_WORKS"}'
</pre>

## Once playbook work you can start building operator

# Build operator
## Install sdk
<pre>
# install : https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md#install-from-homebrew-macos
brew install operator-sdk
</pre>

## Build project
<pre>
operator-sdk new httpd-operator --api-version=httpd.saberkan.fr/v1 --kind=Httpd --type=ansible
</pre>

## Copy role
<pre>
cp roles/httpd_openshift/tasks/main.yml httpd-operator/roles/httpd/tasks/
cd playbook.yaml httpd-operator/playbook.yaml
</pre>

## Edit watcher
<pre>
cat <<EOF > watches.yaml 
---
- version: v1
  group: httpd.saberkan.fr
  kind: Httpd
  playbook: /opt/ansible/playbook.yaml
EOF
cp watches.yaml httpd-operator/watches.yaml
</pre>

## Edit docker file /!\ Do this step manually not use cat
<pre>
cat <<EOF > Dockerfile
FROM quay.io/operator-framework/ansible-operator:v0.10.0

COPY watches.yaml ${HOME}/watches.yaml
COPY playbook.yaml ${HOME}/playbook.yaml
COPY roles/ ${HOME}/roles/
EOF
cp Dockerfile httpd-operator/build/Dockerfile
</pre>

## Build the operator
<pre>
cd httpd-operator
operator-sdk build quay.io/saberkan/httpd-operator:cp-demo
</pre>

## Deploy the operator
<pre>
# Set REPLACE_IMAGE by the builded image
vim deploy/operator.yaml

# Create role
cat <<EOF > role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: httpd-operator
  namespace: operators
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - services
  - endpoints
  - persistentvolumeclaims
  - configmaps
  - secrets
  verbs:
  - create
  - update
  - delete
  - get
  - list
  - watch
  - patch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
- apiGroups:
  - route.openshift.io
  resources:
  - routes
  verbs:
  - create
  - update
  - delete
  - get
  - list
  - watch
  - patch
- apiGroups:
  - ""
  - extensions
  resources:
  - deployments
  - replicasets
  - deployments/finalizers
  verbs:
  - '*'
- apiGroups:
  - ""
  - apps.openshift.io
  resources:
  - deploymentconfigs
  verbs:
  - '*'
- apiGroups:
  - httpd.saberkan.fr
  resources:
  - httpds
  - httpds/status
  verbs:
  - create
  - update
  - delete
  - get
  - list
  - watch
  - patch- "*"
EOF
oc create -f role.yaml

# Create cluster role
cat <<EOF > cluster_role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: httpd-admin-rules
rules:
- apiGroups:
  - httpd.saberkan.fr
  resources:
  - httpds
  - httpds/status
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
EOF
oc create -f cluster_role.yaml

# Create service account
oc create -f deploy/service_account.yaml

# Create role binding
oc create -f deploy/role_binding.yaml 

# Create CRD
oc create -f deploy/crds/httpd_v1_httpd_crd.yaml

# Publish docker image
docker push quay.io/saberkan/httpd-operator:cp-demo

# Deploy operator
oc create -f deploy/operator.yaml

# check logs of opertor pod, A role was missing (not to do in prod env)
oc adm policy add-role-to-user cluster-admin system:serviceaccount:operators:httpd-operator
</pre>

# Create httpd
<pre>
echo 'apiVersion: httpd.saberkan.fr/v1
kind: Httpd
metadata:
  name: example-httpd
spec:
  my_var_value: "IT WORKS WITH OPERATOR"
  size: 1' > httpd.yaml
oc create -f httpd.yaml
</pre>

