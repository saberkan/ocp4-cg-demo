Source:
https://github.com/saberkan/tekton-tutorial
https://openshift.github.io/pipelines-docs/docs/0.8/assembly_using-pipelines.html

# Presentation
## Outil de pipeline approche serverless et cloud native
## Basé sur des tasks, pipelines, et pipelines resources
## Integré à openshift en tant que tech preview
## Possiblité de mettre en place des triggers approche web hooks

# Installation sur OCP 4
<pre>
Operators > OperatorHub > OpenShift Pipelines Operator
Utiliser All namespaces on the cluster (default), automatic
</pre>

# Install CLI
https://github.com/tektoncd/cli

# Install IHM
https://github.com/tektoncd/dashboard

# Utilisation
## Préparation namespace
<pre>
oc new-project pipelines-tutorial
oc get serviceaccount pipeline
</pre>

PS: Des taches pre-configures peuvent etre importés et reutilisés depuis https://github.com/tektoncd/catalog
https://raw.githubusercontent.com/openshift/tektoncd-catalog/release-v0.7/openshift-client/openshift-client-task.yaml
https://raw.githubusercontent.com/openshift/pipelines-catalog/release-v0.7/s2i-java-8/s2i-java-8-task.yaml

## Create maven build task
<pre>
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: maven-build
spec:
  inputs:
    resources:
    - name: workspace-git
      targetPath: /
      type: git
  steps:
  - name: build
    image: maven:3.6.0-jdk-8-slim
    command:
    - /usr/bin/mvn
    args:
    - install
</pre>

## Import OCP cli task
<pre>
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: openshift-client
spec:
  inputs:
    params:
      - name: ARGS
        description: The OpenShift CLI arguments to run
        type: array
        default:
        - "help"
  steps:
    - name: oc
      image: quay.io/openshift/origin-cli:latest
      command: ["/usr/bin/oc"]
      args:
        - "$(inputs.params.ARGS)"
</pre>

## import java s2i task
<pre>
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: s2i-java-8
spec:
  inputs:
    resources:
      - name: source
        type: git
    params:
      - name: PATH_CONTEXT
        description: The location of the path to run s2i from
        default: .
        type: string
      - name: TLSVERIFY
        description: Verify the TLS on the registry endpoint (for push/pull to a non-TLS registry)
        default: "true"
        type: string
      - name: MAVEN_ARGS_APPEND
        description: Additional Maven arguments
        default: ""
        type: string
      - name: MAVEN_CLEAR_REPO
        description: Remove the Maven repository after the artifact is built
        default: "false"
        type: string
      - name: MAVEN_MIRROR_URL
        description: The base URL of a mirror used for retrieving artifacts
        default: ""
        type: string
  outputs:
    resources:
      - name: image
        type: image
  steps:
    - name: gen-env-file
      image: quay.io/openshift-pipeline/s2i
      workingdir: /env-params
      command:
        - '/bin/sh'
        - '-c'
      args:
        - |-
          echo "MAVEN_CLEAR_REPO=$(inputs.params.MAVEN_CLEAR_REPO)" > env-file

          [[ '$(inputs.params.MAVEN_ARGS_APPEND)' != "" ]] &&
            echo "MAVEN_ARGS_APPEND=$(inputs.params.MAVEN_ARGS_APPEND)" >> env-file

          [[ '$(inputs.params.MAVEN_MIRROR_URL)' != "" ]] &&
            echo "MAVEN_MIRROR_URL=$(inputs.params.MAVEN_MIRROR_URL)" >> env-file

          echo "Generated Env file"
          echo "------------------------------"
          cat env-file
          echo "------------------------------"
      volumeMounts:
        - name: envparams
          mountPath: /env-params
    - name: generate
      image: quay.io/openshift-pipeline/s2i
      workingdir: /workspace/source
      command:
        - 's2i'
        - 'build'
        - '$(inputs.params.PATH_CONTEXT)'
        - 'registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift'
        - '--image-scripts-url'
        - 'image:///usr/local/s2i'
        - '--as-dockerfile'
        - '/gen-source/Dockerfile.gen'
        - '--environment-file'
        - '/env-params/env-file'
      volumeMounts:
        - name: gen-source
          mountPath: /gen-source
        - name: envparams
          mountPath: /env-params
    - name: build
      image: quay.io/buildah/stable
      workingdir: /gen-source
      command: ['buildah', 'bud', '--tls-verify=$(inputs.params.TLSVERIFY)', '--layers', '-f', '/gen-source/Dockerfile.gen', '-t', '$(outputs.resources.image.url)', '.']
      volumeMounts:
        - name: varlibcontainers
          mountPath: /var/lib/containers
        - name: gen-source
          mountPath: /gen-source
      securityContext:
        privileged: true
    - name: push
      image: quay.io/buildah/stable
      command: ['buildah', 'push', '--tls-verify=$(inputs.params.TLSVERIFY)', '$(outputs.resources.image.url)', 'docker://$(outputs.resources.image.url)']
      volumeMounts:
        - name: varlibcontainers
          mountPath: /var/lib/containers
      securityContext:
        privileged: true
  volumes:
    - name: varlibcontainers
      emptyDir: {}
    - name: gen-source
      emptyDir: {}
    - name: envparams
      emptyDir: {}
</pre>

## Lister les tasks
<pre>
tkn task ls
oc get tasks
</pre>

## Creation de resources de pipeline input git
<pre>
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: petclinic-git
spec:
  type: git
  params:
  - name: url
    value: https://github.com/spring-projects/spring-petclinic
</pre>


## Creation de resources de pipeline output image 
<pre>
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: petclinic-image
spec:
  type: image
  params:
  - name: url
    value: image-registry.openshift-image-registry.svc:5000/pipelines-tutorial/spring-petclinic
</pre>

## Vérifier les resources
<pre>
tkn resource ls 
oc get pipelineresource
</pre>

## Création de la pipeline
<pre>
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: petclinic-deploy-pipeline
spec:
  resources:
  - name: app-git
    type: git
  - name: app-image
    type: image
  tasks:
  - name: build
    taskRef:
      name: s2i-java-8
    params:
      - name: TLSVERIFY
        value: "false"
    resources:
      inputs:
      - name: source
        resource: app-git
      outputs:
      - name: image
        resource: app-image
  - name: deploy
    taskRef:
      name: openshift-client
    runAfter:
      - build
    params:
    - name: ARGS
      value: "rollout latest spring-petclinic"
/!\ Attention cet argument ne passe pas, attendu sous format d'array par la task
à changer par ["rollout", ...]
</pre>

## Vérifier la pipeline
<pre>
tkn pipeline ls 
oc get pipeline
</pre>

## Lancement de la pipeline
<pre>
tkn pipeline start petclinic-deploy-pipeline \
        -r app-git=petclinic-git \
        -r app-image=petclinic-image \
        -s pipeline

# Vérifier avancement avec output de commande
tkn pipelinerun logs petclinic-deploy-pipeline-run-j48x4 -f -n pipelines-tutorial
</pre>

## Vérifier lancement sur openshift
<pre>
oc get pods
oc get pipelinerun
tkn pipelinerun ls
tkn pipeline describe petclinic-deploy-pipeline
</pre>

## Debuger tasks
<pre>
# Voir logs 
tkn taskrun logs petclinic-deploy-pipeline-run-hj2kb-build-5742n
oc get pods
oc log ..
# Fixer les limits ranges
oc edit limitrange pipelines-tutorial-core-resource-limits
</pre>

## Perspective developpeur
<pre>
oc adm policy add-scc-to-user privileged -z default
oc adm policy add-role-to-user edit -z default
# Ensuite recharge la page et voir rubrique pipeline dans le menu developpeur
</pre>