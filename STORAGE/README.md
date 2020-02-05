Source: https://docs.openshift.com/container-platform/4.3/storage/understanding-persistent-storage.html

# Petit rappel, type de stockage
## Objet comme S3
## Block comme les disks azure
## file systeme comme NFS SMB etc
## LE PV est une notion cluster wide, PVC c'est le "bound" d'un PV dans un namespace. 
## un PV bounded par PVC, ne peut etre utilisé que dans le namespace en question

# La notion de storage class peut se présenter comme ceci
<pre>
Founrisseur de stockage <-> API <-> Storage class (sc A)
   |
   |
   PV (sc A) <-> PVC (sc A) <-> NODE <-> POD (dc, etcd)
</pre>

# Les modes d'acces
Les modes d'accées sont propre au options disponibles par la technologie et type du stockage
## RWO
## RWX
## RWM

# Provisioning des volumes
## Manuel:  Exemple de PV qui point directement vers un serveur NFS sans storage class (provision manuelle)
<pre>
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001
  annotations:
    volume.beta.kubernetes.io/mount-options: rw,nfsvers=4,noexec 
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /tmp
    server: 172.17.0.2
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    name: claim1
    namespace: default

# Pour pouvoir l'utiliser va falloir crée un PVC qui permettra de le monter dans le pod
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc  
spec:
  accessModes:
  - ReadWriteMany      
  resources:
     requests:
       storage: 1Gi

# S'il n'ya pas de SC default, le pvc va chercher un PV de 1G ou plus si disponible et va le "bound"
oc set volume dc/myapp --add --name=v1 -t pvc --claim-name=pvc1 --mount-path=/var/lib/myapp

# Verifier
oc get dc/myapp -o yaml
      volumes:
      - name: v1
        persistentVolumeClaim:
          claimName: pvc1

# La ou il est monté
        volumeMounts:
        - mountPath: /var/lib/myapp
          name: v1

# Scal et vérifier comportement
</pre>

## Automatique: Avec storage class sur AWS
<pre>
# Montage des volumes dans le pod à travers dc
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
parameters:
  encrypted: "true"
  type: gp2
provisioner: kubernetes.io/aws-ebs
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer

# Ensuite il suffit de crée un PVC, Kubernetes va crée le PV requis et le bound au PVC
</pre>

# Claim policy
## Retain : Quand le PVC est supprimé, le PV Reste
## Delete : Quand le PVC est supprimé le PV l'est aussi.