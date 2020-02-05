Source: https://docs.openshift.com/container-platform/4.3/backup_and_restore/disaster_recovery/scenario-2-restoring-cluster-state.html

# Se connecter sur un master
<pre>
sudo /usr/local/bin/etcd-snapshot-backup.sh ./assets/backup
</pre>

# Supprimer le failed master du cluster
Supposons master1 down
## Se connecter à un master up
<pre>
ssh master0
sudo -E etcd-member-remove.sh master1
</pre>

## Vérifier que le node a été supprimé du cluster etcd
<pre>
# S'y connecter
id=$(sudo crictl ps --name etcd-member | awk 'FNR==2{ print $1}') && sudo crictl exec -it $id /bin/sh

# Setter les variables ETCD
export ETCDCTL_API=3 ETCDCTL_CACERT=/etc/ssl/etcd/ca.crt ETCDCTL_CERT=$(find /etc/ssl/ -name *peer*crt) ETCDCTL_KEY=$(find /etc/ssl/ -name *peer*key)

# Lister le memberes
etcdctl member list -w table
</pre>

# Ajouter le nouveau master au cluster
<pre>
# S'y connecter
ssh master1

# L'ajouter
sudo -E etcd-member-add.sh <IP D'UN MEMBRE ETCD EXISTANT> master1

# Vérifier 3 pods etcd
oc get pods -n openshift-etcd

# Vérifier que l'etcd est bien ajouté au cluster
# S'y connecter
id=$(sudo crictl ps --name etcd-member | awk 'FNR==2{ print $1}') && sudo crictl exec -it $id /bin/sh

# Setter les variables ETCD
export ETCDCTL_API=3 ETCDCTL_CACERT=/etc/ssl/etcd/ca.crt ETCDCTL_CERT=$(find /etc/ssl/ -name *peer*crt) ETCDCTL_KEY=$(find /etc/ssl/ -name *peer*key)

etcdctl member list -w table
etcdctl endpoint health --cluster
</pre>
