# Forker un projet example pour crée des versions différentes
Nous allons utiliser https://github.com/sclorg/ruby-ex.git
Forker sous : https://github.com/saberkan/ruby-ex.git

# Cloner le projet
<pre>
git clone https://github.com/sclorg/ruby-ex.git
cd ruby-ex
</pre>

# Créer la branche A
<pre>
git checkout -b A 
# Ensuite éditer le contenu à afficher pour reconnaitre la branche au deploiement : config.ru
git add .
git commit -m "Adding version"
git push origin A
</pre>

# Créer la branche B
<pre>
git checkout -b B
# Ensuite éditer le contenu à afficher pour reconnaitre la branche au deploiement : config.ru
git add .
git commit -m "Adding version"
git push origin B
</pre>

# Deployer l'application à partir
<pre>
oc new-project ab-testing
oc new-app centos/ruby-25-centos7~https://github.com/saberkan/ruby-ex.git --name ruby-a
occ new-app centos/ruby-25-centos7~https://github.com/saberkan/ruby-ex.git --name ruby-b
</pre>

# Crée l'image stream ruby-ex pour recevoir les images buildés
<pre>
oc create is ruby-ex
</pre>

# Editer les build config pour build la bonne branche, exemple pour A (faire la meme chose pour B)
<pre>
oc edit bc/ruby-a
# Ajout ref pour choisir la branche
    git:
      uri: 'https://github.com/saberkan/ruby-ex.git'
      ref: "A"
# Changer le tag de l'image à générer
  output:
    to:
      kind: ImageStreamTag
      name: 'ruby-ex:A'
</pre>

# Editer les deploiement config pour pointer vers la bonne version de l'image, exemple pour A (faire la meme chose pour B)
<pre>
oc edit dc/ruby-a
        from:
          kind: ImageStreamTag
          namespace: ab-testing
          name: 'ruby-ex:A'
</pre>

# Relancer les builds
<pre>
oc start-build ruby-a
oc start-build ruby-b
</pre>

# Vérifier que les nouvelles images sont crées
<pre>
oc describe is ruby-ex
...
A
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/ab-testing/ruby-ex@sha256:9c507bc9d4af85d99d87ee45d794e9ccdb495d09524934bcf885652dac893635
      3 seconds ago

B
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/ab-testing/ruby-ex@sha256:f78160de99cc64296882b08e5c69ca2efdb94b626e1775ce2aabe9abef44f7a3
      5 seconds ago
</pre>

# Le déploiement se déclenchent automatiquement parcequ'ils sont configurés avec des triggers

# Exposer les services
<pre>
oc expose svc/ruby-a
oc expose svc/ruby-b
</pre>

# Vérifier que les deux routes retournent la bonne version du code
<pre>
oc get routes
NAME     HOST/PORT                                                              PATH   SERVICES   PORT       TERMINATION   WILDCARD
ruby-a   ruby-a-ab-testing.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com          ruby-a     8080-tcp                 None
ruby-b   ruby-b-ab-testing.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com          ruby-b     8080-tcp                 None

curl -X GET ruby-a-ab-testing.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com | grep 'VERSION A'
curl -X GET ruby-b-ab-testing.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com | grep 'VERSION B'
</pre>

# Crée une route unique qui redirige 10% de traffic à B, et 90% à A
<pre>
cat <<EOF > route.yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: ruby-ab-testing
  namespace: ab-testing
spec:
  host: >-
    ruby-ab-testing-ab-testing.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com
  to:
    kind: Service
    name: ruby-a
    weight: 90
  alternateBackends:
    - kind: Service
      name: ruby-b
      weight: 10
  port:
    targetPort: 8080-tcp
  wildcardPolicy: None
EOF
oc create -f route.yaml
</pre>

# Vérifier en lancant plusieurs fois la commande suivante:
<pre>
curl ruby-ab-testing-ab-testing.apps.cluster-mpl-d015.mpl-d015.example.opentlc.com | grep -i version
</pre>

# Attention par defaut la route a une session affinity, ainsi, avec le navigateurs on reste sur la meme version que la premiere connexion