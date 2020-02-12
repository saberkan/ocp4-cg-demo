# Préparatoin
## Scale up 3 nouveau nodes (64GRAM, 16CPU) + 120G disk route

# Instalation
## Création namespace
<pre>
oc create namespace openshift-storage
oc label namespace openshift-storage "openshift.io/cluster-monitoring=true"
oc get -n openshift-console route console
</pre>

## Install
Sur la console
Operator hub > OpenShift Container Storage Operator > install > namespace: openshift-storage > Selectionner les nodes crée pour cet objectif

## Attendre jusqu'a l'install up
watch oc -n openshift-storage get csv

## Ceci va crée 3 disks de 2T et les monter chaqu'un sur un node, et 3 disks de 10G et les monter chaqu'un sur un node

## Les 6T sont distinées au données, et 30G au monitoring

## Il faut trouver une liste de pods crées comme ceci
<pre>
oc -n openshift-storage get pods
NAME                                                              READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-6t5cc                                            3/3     Running     0          9m30s
csi-cephfsplugin-7slmp                                            3/3     Running     0          9m30s
csi-cephfsplugin-865k6                                            3/3     Running     0          9m30s
csi-cephfsplugin-8zn2w                                            3/3     Running     0          9m30s
csi-cephfsplugin-9mmkp                                            3/3     Running     0          9m30s
csi-cephfsplugin-provisioner-57f9567c-g44d6                       4/4     Running     0          9m30s
csi-cephfsplugin-provisioner-57f9567c-tlnjz                       4/4     Running     0          9m30s
csi-cephfsplugin-q86tr                                            3/3     Running     0          9m30s
csi-rbdplugin-24t87                                               3/3     Running     0          9m30s
csi-rbdplugin-4zp5v                                               3/3     Running     0          9m30s
csi-rbdplugin-5s5dc                                               3/3     Running     0          9m30s
csi-rbdplugin-fjl6s                                               3/3     Running     0          9m30s
csi-rbdplugin-mrkr5                                               3/3     Running     0          9m30s
csi-rbdplugin-pr9hn                                               3/3     Running     0          9m30s
csi-rbdplugin-provisioner-69bb78d655-4hrzx                        4/4     Running     0          9m30s
csi-rbdplugin-provisioner-69bb78d655-clzk5                        4/4     Running     0          9m30s
noobaa-core-0                                                     2/2     Running     0          5m38s
noobaa-operator-6d59cd7855-lhwg8                                  1/1     Running     0          12m
ocs-operator-dc48d685-8qtqn                                       1/1     Running     0          12m
rook-ceph-drain-canary-731c9dd8472082ee17090f034387aa3b-78k55k9   1/1     Running     0          5m48s
rook-ceph-drain-canary-93e6590ed0bb3c88b985beb159ef084a-7c8kpsg   1/1     Running     0          5m55s
rook-ceph-drain-canary-d0eaaceda4b757fe363def0873a5f86f-98s5k27   1/1     Running     0          5m45s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-7c865846g6k27   1/1     Running     0          5m33s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-69f846986g4n7   1/1     Running     0          5m32s
rook-ceph-mgr-a-5486fc7cf5-l9h6z                                  1/1     Running     0          6m44s
rook-ceph-mon-a-58c7cd4f9b-g4z2m                                  1/1     Running     0          8m17s
rook-ceph-mon-b-684f66b8df-992wc                                  1/1     Running     0          7m43s
rook-ceph-mon-c-6f7657b8b6-2sxl5                                  1/1     Running     0          7m13s
rook-ceph-operator-d857c476f-4npzl                                1/1     Running     0          12m
rook-ceph-osd-0-8675cf4f4-7gpbv                                   1/1     Running     0          5m55s
rook-ceph-osd-1-58b9d954cf-9s6bw                                  1/1     Running     0          5m48s
rook-ceph-osd-2-6994dd5f44-hsqrv                                  1/1     Running     0          5m45s
rook-ceph-osd-prepare-ocs-deviceset-0-0-d2ppm-vvlt8               0/1     Completed   0          6m22s
rook-ceph-osd-prepare-ocs-deviceset-1-0-9tmc6-svb84               0/1     Completed   0          6m22s
rook-ceph-osd-prepare-ocs-deviceset-2-0-qtbfv-j4nr4               0/1     Completed   0          6m21s
</pre>

## Utilisation
3 storage class sont crée, filesystem, block et object
<pre>
oc get sc
ocs-storagecluster-cephfs
ocs-storagecluster-ceph-rbd
openshift-storage.noobaa.io
</pre>

## debuggage
<pre>
curl -s https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/toolbox.yaml | sed 's/namespace: rook-ceph/namespace: openshift-storage/g'| oc apply -f -

TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD

ceph status
ceph osd status
ceph osd tree
ceph df
rados df
ceph versions
</pre>