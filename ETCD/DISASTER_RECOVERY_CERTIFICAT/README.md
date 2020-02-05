# Recupération du cluster quand les certificats sont expirés. Les certificats sont renouvlés chaque 24, si le cluster se trouve dans une sitution ou il a pas pu mettre à jour les certifs (shutdown). Cette procédure doit etre suivi.

## Création d'un serveru API pour recovry
<pre>
Se connecter sur un master (celui ou le certificats est expiré)
Obtenir l'image release (ex: 4.3.0), peut etre trouver dans les images sur le nodes
RELEASE_IMAGE=<release_image>

# Obtenir version kube api operator
KAO_IMAGE=$( oc adm release info --registry-config='/var/lib/kubelet/config.json' "${RELEASE_IMAGE}" --image-for=cluster-kube-apiserver-operator )

# Recupérer l'image
podman pull --authfile=/var/lib/kubelet/config.json "${KAO_IMAGE}"

# Crée un nouveau server api pour recupérer le cluster
podman run -it --network=host -v /etc/kubernetes/:/etc/kubernetes/:Z --entrypoint=/usr/bin/cluster-kube-apiserver-operator "${KAO_IMAGE}" recovery-apiserver create

# Recupérer le fichier kubeconfig, là ou il est
export KUBECONFIG=admin.kubeconfig

# Vérifier le démarrage de recovery server
until oc get namespace kube-system 2>/dev/null 1>&2; do echo 'Waiting for recovery apiserver to come up.'; sleep 1; done
</pre>

## Regénérer les certifs
<pre>
podman run -it --network=host -v /etc/kubernetes/:/etc/kubernetes/:Z --entrypoint=/usr/bin/cluster-kube-apiserver-operator "${KAO_IMAGE}" regenerate-certificates
</pre>

## Redeploiement
<pre>
# Forcer redeploiement du controle plane
oc patch kubeapiserver cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge
oc patch kubecontrollermanager cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge
oc patch kubescheduler cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge

# Crée le bootstrap
recover-kubeconfig.sh > kubeconfig

# Copier le sur les 3 masters vers
/etc/kubernetes/kubeconfig

# Recupérer le ca
oc get configmap kube-apiserver-to-kubelet-client-ca -n openshift-kube-apiserver-operator --template='{{ index .data "ca-bundle.crt" }}' > /etc/kubernetes/kubelet-ca.crt

# Copier le sur les 3 masters vers
/etc/kubernetes/kubelet-ca.crt

# Forcer le deamon machine set a accepter le cetif
touch /run/machine-config-daemon-force

# Recupérer le kublet sur tous les masters
systemctl stop kubelet
rm -rf /var/lib/kubelet/pki /var/lib/kubelet/kubeconfig
ystemctl start kubelet

# Refaire pareil sur les nodes failed
# Accepter les requests en pending
oc get csr
oc describe csr <csr_name> 
oc adm certificate approve <csr_name>
</pre>

## Supprimer la recovery API
<pre>
podman run -it --network=host -v /etc/kubernetes/:/etc/kubernetes/:Z --entrypoint=/usr/bin/cluster-kube-apiserver-operator "${KAO_IMAGE}" recovery-apiserver destroy
</pre>

