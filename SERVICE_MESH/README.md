Source:
https://istio.io/docs/tasks/traffic-management/
Livre d'intro (sur ocp3) :
Introducing istio service mesh par Christian Posta & Burr Sutter

Des labs demo:
https://learn.openshift.com/servicemesh?extIdCarryOver=true&intcmp=7013a000002CtetAAC&sc_cid=701f2000001Css5AAC

# Evolution services micro (Théorie)
Low risk monolith to micro service process
<pre>
https://developers.redhat.com/blog/2017/09/26/low-risk-monolith-microservice-evolution-part/
</pre>

# Service Mesh
## Interet
- Tracing
- Circuit breaker
- Routing
- SVC discovery
- Load balancing
- Failure recovery
- a/b testing
- Canary release
- Rate limits
- Access control
- End to end authentication

## Istio
- Envoy proxy: Ajouter capacités itstio au deploiement existant (sidecard)
- Mixer: collecter les metrics depuis envoy, evaluation et access control
- Pilot: service discovry, Traffic management capacities, et commnication aux sidecards
- Auth/Citadel: authentication 

## Elasticsearch
## Jaeger
## Kiali

# Installation
## Install elasticsearch
Install from operator hub on all cluster

## Install Jaeger
Install from operator hub on all cluster

## Install Kiali
Install from operator hub on all cluster

## Check install
<pre>
oc get ClusterServiceVersion | grep -i elastic
oc get ClusterServiceVersion | grep -i jaeger
oc get ClusterServiceVersion | grep kiali
oc get pod  -n openshift-operators
</pre>

## Wait unitl all succeeded

## Install service mesh operator
<pre>
oc adm new-project istio-operator --display-name="Service Mesh Operator"
oc project istio-operator
oc apply -n istio-operator -f https://raw.githubusercontent.com/Maistra/istio-operator/maistra-1.0.0/deploy/servicemesh-operator.yaml
oc get pod -n istio-operator
oc logs -n istio-operator $(oc -n istio-operator get pods -l name=istio-operator --output=jsonpath={.items..metadata.name})
</pre>

## Create a controle plane
<pre>
oc adm new-project istio-system --display-name="Service Mesh System"
cat <<EOF > servicemesh.yaml
apiVersion: maistra.io/v1
kind: ServiceMeshControlPlane
metadata:
  name: service-mesh-installation
spec:
  threeScale:
    enabled: false

  istio:
    global:
      mtls: false
      disablePolicyChecks: false
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 128Mi

    gateways:
      istio-egressgateway:
        autoscaleEnabled: false
      istio-ingressgateway:
        autoscaleEnabled: false
        ior_enabled: false

    mixer:
      policy:
        autoscaleEnabled: false

      telemetry:
        autoscaleEnabled: false
        resources:
          requests:
            cpu: 100m
            memory: 1G
          limits:
            cpu: 500m
            memory: 4G

    pilot:
      autoscaleEnabled: false
      traceSampling: 100.0

    kiali:
      dashboard:
        user: admin
        passphrase: redhat
    tracing:
      enabled: true
EOF
oc apply -f servicemesh.yaml -n istio-system
oc get pods -n istio-system
</pre>

## Get Kiali route
<pre>
oc get routes -n istio-system
# wait until route created
oc get route kiali -n istio-system -o jsonpath='{"https://"}{.spec.host}{"\n"}'
</pre>

## Activer servicemesh sur notre namespace projet
<pre>
cat <<EOF > memberpoll.yaml
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
spec:
  members:
  - user1-tutorial
EOF
oc create -f memberpoll.yaml -n istio-system
</pre>


# Utilisation
## Présentation
VirtualService :	comment router la requete
DestinationRule : 	politique de routing apres virtualservice
ServiceEntry: Activer requetes à l'extérieur de service mesh
Gateway	: Configuration de load balancing
• HTTP/TCP ingress traffic to mesh application
• Egress traffic to external services
Sidecar	: Proxies devant applications


## Création projet
gateway -> partner -> catalog
<pre>
export OCP_TUTORIAL_PROJECT=user1-tutorial
oc new-project $OCP_TUTORIAL_PROJECT
git clone https://github.com/gpe-mw-training/ocp-service-mesh-foundations
cd ocp-service-mesh-foundations

# Deployer catalog
cd catalog
oc create -f kubernetes/catalog-service-template.yml -n $OCP_TUTORIAL_PROJECT
oc create -f kubernetes/Service.yml -n $OCP_TUTORIAL_PROJECT
# Check created pod that it has envoy proxy

# Deployer partner
cd ../partner
oc create -f kubernetes/partner-service-template.yml -n $OCP_TUTORIAL_PROJECT
oc create -f kubernetes/Service.yml -n $OCP_TUTORIAL_PROJECT

# Deployer gateway
cd ../gateway
oc create -f kubernetes/gateway-service-template.yml -n $OCP_TUTORIAL_PROJECT
oc create -f kubernetes/Service.yml -n $OCP_TUTORIAL_PROJECT
# Check created pod that it has envoy proxy

# Check all works
oc expose svc gateway
# Acces the route
</pre>

## Créate service mesh routing
<pre>
cat <<EOF > routing.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - '*'
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ingress-gateway
spec:
  hosts:
  - '*'
  gateways:
  - ingress-gateway
  http:
  - match:
    - uri:
        exact: /
    route:
    - destination:
        host: gateway
        port:
          number: 8080
EOF
oc apply -f routing.yaml -n $OCP_TUTORIAL_PROJECT

# Vérifier accessibilité via istio
export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')
curl $GATEWAY_URL
</pre>



## Dynamique routing
catalog v1 et v2
<pre>
cd ../catalog-v2
oc create -f kubernetes/catalog-service-template.yml -n $OCP_TUTORIAL_PROJECT
oc get pods -l application=catalog -n $OCP_TUTORIAL_PROJECT -w

# Le service par default redirige vers les 2 options
oc describe service catalog -n $OCP_TUTORIAL_PROJECT | grep Selector
oc get deploy catalog-v1 -o json -n $OCP_TUTORIAL_PROJECT | jq .spec.template.metadata.labels
oc get deploy catalog-v2 -o json -n $OCP_TUTORIAL_PROJECT | jq .spec.template.metadata.labels

# Test
curl $GATEWAY_URL
# Une fois sur la v1 et l'autre sur v2

# Rediriger tout vers v2
cat <<EOF > v2.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v2
      weight: 100
EOF
oc apply -f v2.yaml -n $OCP_TUTORIAL_PROJECT

cat <<EOF > rule.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  creationTimestamp: null
  name: catalog
spec:
  host: catalog
  subsets:
  - labels:
      version: v1
    name: version-v1
  - labels:
      version: v2
    name: version-v2
EOF
oc create -f rule.yaml -n $OCP_TUTORIAL_PROJECT

# Test ou meme curl sur la route de l'application
curl $GATEWAY_URL
# Toujours v2

# ouvrir kiali pour voir les flux
</pre>

# Canary
<pre>
cat <<EOF > canary.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
    - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
      weight: 80
    - destination:
        host: catalog
        subset: version-v2
      weight: 20
EOF
oc apply -f canary.yaml -n $OCP_TUTORIAL_PROJECT

curl -H $GATEWAY_URL
</pre>


# Headers routing Seule le header user-agent Safari a le droit d'aller à la v2
<pre>
cat <<EOF > headers.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - match:
    - headers:
        user-agent:
          regex: .*Safari.*
    route:
    - destination:
        host: catalog
        subset: version-v2
  - route:
    - destination:
        host: catalog
        subset: version-v1
EOF
oc apply -f headers.yaml -n $OCP_TUTORIAL_PROJECT

while true
do curl ${GATEWAY_URL}
sleep .1
done

while true
do curl -H'user-agent: Safari' catalog-user1-tutorial.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com
sleep .1
done
</pre>


# Mirroring
<pre>
cat <<EOF > mirror.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  gateways:
  - ingress-gateway
  hosts:
    - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
      weight: 100
    mirror:
      host: catalog
      subset: version-v2
    mirror_percent: 100
EOF
oc apply -f mirror.yaml -n $OCP_TUTORIAL_PROJECT

while true
do curl ${GATEWAY_URL}
sleep .1
done
</pre>

# Circuit breaker //TODO
Fails uand on depasse une requete en parallere http1MaxPendingRequests: 1, maxRequestsPerConnection: 1
<pre>
cat <<EOF > circuit-breaker.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker
spec:
  host: catalog
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
      tcp:
        maxConnections: 1
    outlierDetection:
      baseEjectionTime: 180.000s
      consecutiveErrors: 1
      interval: 1.000s
      maxEjectionPercent: 100
EOF
oc create -f circuit-breaker.yaml -n $OCP_TUTORIAL_PROJECT
</pre>

# D'autre stratégies de redirection peuvent etre retrouvés dans la documentation officielle de istio
https://istio.io/docs/tasks/traffic-management/
