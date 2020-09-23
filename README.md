# kumoscale-csi-plugin

The ks-csi-plugin enables storage provisioning from Kubernetes(R) on top of KumoScaleTM and provisioner.
See the ks-csi.yaml on how to deploy on Kubernetes cluster.
You also need to update the versions of the ks-csi-plugin image.
The current version of ks-clsi-plugin requires Kubernetes version >= 14.1.
Use ks-csi.yaml for kube version >= 1.16 and ks-csi15.yaml for kube version which is < 1.16 .

# Kubernetes install preparation

1) Download ks-csi-plugin-v1.4.244.tar.gz
2) Upload this driver image file to local container registry with appropriate tag

# Kubernetes install instructions

1) Download the yaml example artifacts
2) Edit the provisioner-secret.yaml and provide the provisioner url , kumoscale-generated token and optionaly tenantId of an existing provisioner tenant.
3) Create the secret:
    kubectl create -f provisioner-secret.yaml
4) Edit the install yaml: ks-csi.yaml with required parameters as explained above.
  If kublet is running with non default root directory (the default root directory is /var/lib/kubelet)
  You need to replace every occurence of /var/lib/kubelet in the instalation yaml with with the root directory which is in use.
5) Install the driver with:
    kubectl create -f ks-csi.yaml

# Kubernetes - Provision storage from KumoScale and provisioner

1) Create a storage class with one of the examples, for example:
    kubectl create -f provisioner-storage-class.yaml
    The CSI plug-in supports the following parameters in the storage class:
    See storage class parameters summary bellow for supported options
2) Create a  persistent volume claim (PVC) with one of the example which is based on the storage class you've created.
3) Create a pod based on the PVC.

# Kubernetes - Expanding volumes

1) Kubernetes 'Expand Volume' is in Beta (1.15) and Alpha (1.14), therefor, it needs to be enabled by
  --feature-gates=ExpandCSIVolumes=true,ExpandInUsePersistentVolumes=true 
  in centos kube  do:
    a) In all kube nodes : Edit /var/lib/kubelet/kubeadm-flags.env :
    Add the above feature gates to the variable KUBELET_KUBEADM_ARGS .
    b) In master do:
    add the feature gates to api server in file : /etc/kubernetes/manifests/kube-apiserver.yaml
2) To expand a volume, the storage class needs to enable it by setting:
  allowVolumeExpansion: true 
  See example in storage class yamls.
3) A special secret should be created with the namespace kube-system and the name kumoscale-provisioner.
  This secret should contain the provisioner url and token.
  You can use the provisioner-secret.yaml from the example for this purpose
4) To expand a volume , you need to edit the PVC:
  kubectl edit pvc <pvc name>
  Simply enlarge the pvc size under spec->storage
  Allow 1-2 minutes for the expansion to occur, as it take some time for kube to notice the change and
  perform the expansion.

# Kubernetes - Snapshots

1) Kubernetes 'Snapshots' is in Alpha (1.16), therefor, it needs to be enabled by
  --feature-gates=VolumeSnapshotDataSource=true
  See 'Expand Volume' section on how to enable feature gates.
2) To create a snapshot, you first need to create a VolumeSnapshotClass - see SnapshotClass.yaml for example.
  You can specify the reservedSpace for snapshots of this class in percentage from the original volume size.
3) Create a PVC with a storage class which has replicas=1 (use kube1PVC.yaml for example)
4) Create a snapshot with a snapshot yaml that refers to the created PVC(See Snapshot.yaml example)
5) Create a snapshot volume with a snapshot volume yaml (See SnapshotVolume.yaml example)
6) Deletion needs to be in the same order (first delete the snashot volume then, the snapshot and then the volume)

# Kubernetes - Storage Class Parameters Summary

  File system type: (supported types ext2,ext3,ext4,xfs default : ext4) :
  #csi.storage.k8s.io/fstype: "xfs"
  
  Number of replicas (default 1):
  #numReplicas: "1"
  
  Allow replicas on KumoScales from the same rack (boolean : default "false"):
  #sameRackAllowed: "false"
  Limit IOPS on volumes - per giga byte (default - "0" - unlimmited )
  #maxIOPSPerGB : "0"
  Desired IOPS for volume - per gig byte (default - "0" - not set)
  #desiredIOPSPerGB: "0"
  Limit bandwidth on volumes - per giga byte, in KB/sec (default - "0" - unlimmited )
  #maxBWPerGB: "0"
  Desired bandwidth on volumes - per giga byte,in KB/sec (default - "0" - not set)
  #desiredBWPerGB: "0"
  Volume block size - Volume and file system block size (default 4096)
  #blockSize: "4096"
  The space to reserve for snapshot volume changes in (%) from from total size
  #default 10
  reservedSpacePercentage: "20"   
  
  Specify whether snapshot volumes from this class are writable or not
  #default true
  writable: "true"
  
  Select thick for thick volume or thin for thin volume - default thick
  #provisioningType: "thick"

# Kubernetes - Collecting Logs

  To collect logs for each container (ks-csi-plugin) you can use kubectl logs.
  For example, to collect controller logs run:
        kubectl logs -n kube-system csi-kumoscale-controller-0 ks-csi-plugin > controller.log

# Kubernetes - CSI topology supported

CSI topology features enables volume allocation on backends which are accessible to nodes.
In order to enable this feature the user needs to:
1) Set volumeBindingMode: WaitForFirstConsumer in the storage class -
So that pvc allocation will be done when a pod that refers it starts running for the first time
2) Label each of Kubernetes nodes with topology labels:
    failure-domain.beta.kubernetes.io/region
    failure-domain.beta.kubernetes.io/zone
  Note: after labeling nodes, you need to redploy the csi driver.
3) Set  region ,zone, rack to backends (use REST command update backend or add backend)
4) Check that when a pod is assigned to a node, the volume is allocated on backend within the same region and zone of the
node when possible (such backend exist with enough capacity)

# CSI Driver

https://github.com/KioxiaAmerica/kumoscale/tree/CSI
