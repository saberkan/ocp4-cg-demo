# Restore etcd on one master from backup
<pre>
export INITIAL_CLUSTER="etcd-member-ip-10-0-143-125.ec2.internal=https://etcd-0.clustername.devcluster.openshift.com:2380"
sudo /usr/local/bin/etcd-snapshot-restore.sh /home/core/snapshot_db_kuberesources_<datetimestamp>.tar.gz $INITIAL_CLUSTER
C
oc get nodes -l node-role.kubernetes.io/master
</pre>


# Create new master hosts then accept thier certificates
<pre>
oc get csr
oc describe csr <csr_name> 
oc adm certificate approve <csr_name>
</pre>

# Update load balancer, and DNS entries

# Set Up temporary signer certificate for etcd
<pre>
export KUBE_ETCD_SIGNER_SERVER=$(sudo oc adm release info --image-for kube-etcd-signer-server --registry-config=/var/lib/kubelet/config.json)
sudo -E /usr/local/bin/tokenize-signer.sh master0
oc create -f assets/manifests/kube-etcd-cert-signer.yaml
ss -ltn | grep 9943
</pre>

# Add second master
<pre>
ssh master1
export SETUP_ETCD_ENVIRONMENT=$(sudo oc adm release info --image-for machine-config-operator --registry-config=/var/lib/kubelet/config.json)
export KUBE_CLIENT_AGENT=$(sudo oc adm release info --image-for kube-client-agent --registry-config=/var/lib/kubelet/config.json)
sudo -E /usr/local/bin/etcd-member-recover.sh <IP_MASTER0> master1
</pre>

# Check all ok
<pre>
id=$(sudo crictl ps --name etcd-member | awk 'FNR==2{ print $1}') && sudo crictl exec -it $id /bin/sh
export ETCDCTL_API=3 ETCDCTL_CACERT=/etc/ssl/etcd/ca.crt ETCDCTL_CERT=$(find /etc/ssl/ -name *peer*crt) ETCDCTL_KEY=$(find /etc/ssl/ -name *peer*key)
etcdctl member list -w table
</pre>	

# Repeat to add master2

# Delete signer pod etcd
<pre>
oc delete pod -n openshift-config etcd-signer
</pre>
