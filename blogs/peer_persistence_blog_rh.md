For nearly two decades, HPE and Red Hat have had a long standing partnership providing jointly supported software, platforms and services including many reference architectures and configurations which support HPE and Red Hat customers in their container initiatives.  Whether you’re building physical, virtual, or cloud environments, you can be confident in our tested solutions and world-class services and support.

The addition of new capabilities the HPE CSI Operator for Kubernetes available in the [Red Hat Ecosystem Catalog](https://catalog.redhat.com/software/operators/detail/5e9874643f398525a0ceb004) and deployed from the [OpenShift OperatorHub](https://operatorhub.io/operator/hpe-csi-operator) never stops. 

## Technology Overview

The HPE CSI Operator for Kubernetes has introduced support for HPE Remote Copy Peer Persistence technology for HPE Primera and 3PAR storage systems to protect the data of your mission critical applications running within Red Hat OpenShift Container platform and other containerized environments. These features come as part of the upstream HPE CSI Driver for Kubernetes that has been developed by HPE and certified by Red Hat. The HPE CSI Driver for Kubernetes is a comprehensive CSI compliant driver per the Kubernetes CSI specification providing important data management capabilities for containerized workloads including but not limited to CSI native dynamic provisioning, snapshots, cloning. Along with the native CSI capabilites, the HPE CSI Driver enables array specific features to be defined within the StorageClass, such as the capability of configuring performance policies or protection templates for persistent volumes on Nimble Storage and enabling HPE Remote Copy Peer Persistence for workloads running on HPE Primera. 

To learn more about the full capabilities of the HPE CSI Driver for Kubernetes check out https://scod.hpedev.io.

## Feature overview - HPE Remote Copy Peer Persistence for Primera 

As more and more applications migrate into Kubernetes, it is important to ensure that mission-critical applications using  are highly available and resistant to failure. Using HPE Remote Copy Peer Persistence with the HPE CSI Operator and Red Hat OpenShift provides enhanced availability and transparent failover for disaster recovery protection with Kubernetes.  HPE Primera and 3PAR Remote Copy can serve as the foundation for a disaster recovery solution.

HPE Remote Copy within Kubernetes provides enhanced availability and transparent failover for disaster recovery protection with Kubernetes. As more and more applications migrate into Kubernetes, HPE recommends customers deploy mission-critical applications with replicated persistent volumes to ensure that these applications are highly available and resistant to failure. HPE Primera and 3PAR Remote Copy can serve as the foundation for a disaster recovery solution. 

For information on creating a Peer Persistence configuration, review the [HPE Primera Peer Persistence Host OS Support Matrix](https://techhub.hpe.com/eginfolib/storage/docs/Primera/RemoteCopy/RCconfig/GUID-1F726F48-A372-4ED8-B1D7-9545D091AE98.html#GUID-1F726F48-A372-4ED8-B1D7-9545D091AE98) for the supported host OSs and host persona requirements. Refer to [HPE Primera OS: Configuring data replication using Remote Copy over IP](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=emr_na-a00088914en_us) for more information.

## Get Started
#### Pre-requisites:

  - Single zone Kubernetes cluster
  - All zoning and Remote Copy links configured (iSCSI or FC) between sites along with Quorum Witness
  - HPE CSI Operator for Kubernetes deployed

### Deploy the HPE CSI Operator for Kubernetes
Here is a guide along with a tutorial video for deploying the HPE CSI Operator for Kubernetes within a Red Hat OpenShift cluster.

 - [HPE CSI Operator for Kubernetes deployment on Red Hat Openshift](https://scod.hpedev.io/partners/redhat_openshift/index.html)
 - [Video tutorial: Install the HPE CSI Operator for Kubernetes on Red Hat OpenShift](https://www.youtube.com/watch?v=tBjjGuuOn7Q)

### Create `Secret` for Remote Copy links
Once the HPE CSI Driver is deployed, start by creating two secrets. One for the HPE Primera array at each site (i.e. default-primera-secret and secondary-primera-secret) that are part of the Remote Copy links. 

#### Primary Array

```markdown
apiVersion: v1
kind: Secret
metadata:
  name: default-primera-secret
  namespace: hpe-csi-driver
stringData:
  serviceName: primera3par-csp-svc
  servicePort: "8080"
  backend: 10.0.0.2
  username: 3paradm
  password: 3pardata
```  
**Note:** Verify that the `Namespace` defined within the `Secret` is the same as the OpenShift `Project` name used when deploying the HPE CSI Operator.

#### Secondary Array

```markdown
apiVersion: v1
kind: Secret
metadata:
  name: secondary-primera-secret
  namespace: hpe-csi-driver
stringData:
  serviceName: primera3par-csp-svc
  servicePort: "8080"
  backend: 10.0.0.3
  username: 3paradm
  password: 3pardata
```  

## Create Peer Persistence CustomResourceDefinition

The next step would be to create a `CustomResourceDefinition` or `CRD` to hold the target array information that will be used when creating the volume pairs. 

```markdown
apiVersion: storage.hpe.com/v1
kind: HPEReplicationDeviceInfo
metadata:
  name: replication-crd
spec:
  target_array_details:
  - targetCpg: SSD_r6
    targetName: primera-c670
    targetSecret: secondary-primera-secret
    #targetSnapCpg: SSD_r6 (optional)
    targetSecretNamespace: hpe-csi-driver
```

## Create Peer Persistence StorageClass

With the `Secrets` and `CustomResourceDefinition` available the HPE CSI Operator is now configured and ready to provision replicated volumes. 

To get started provisioning replicated volumes, create a replication enabled `StorageClass` in the primary site using the **remoteCopyGroup: <remote_copy_group_name>** and the **replicationDevices: <replication_crd_name>** parameters. The HPE CSI Driver can use an existing Remote Copy Group or will create a new one based upon the name specified in the `StorageClass`. The CSI Driver will also use the information from the `CRD` to create the replicated volume on the target array.

**Note:** A Remote Copy group is a group of one or more volumes on an HPE Primera array to be replicated to another system. Because the volumes in a Remote Copy group are related, Remote Copy ensures that the data on the volumes within the group maintain write consistency.

For a full list of available `StorageClass` parameters: see [StorageClass Parameters](https://scod.hpedev.io/container_storage_provider/hpe_3par_primera/index.html#storageclass_parameters).

```markdown
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
  name: replicated-storageclass
provisioner: csi.hpe.com
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  csi.storage.k8s.io/fstype: xfs
  csi.storage.k8s.io/controller-expand-secret-name: default-primera-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: hpe-csi-driver
  csi.storage.k8s.io/controller-publish-secret-name: default-primera-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: hpe-csi-driver
  csi.storage.k8s.io/node-publish-secret-name: default-primera-secret
  csi.storage.k8s.io/node-publish-secret-namespace: hpe-csi-driver
  csi.storage.k8s.io/node-stage-secret-name: default-primera-secret
  csi.storage.k8s.io/node-stage-secret-namespace: hpe-csi-driver
  csi.storage.k8s.io/provisioner-secret-name: default-primera-secret
  csi.storage.k8s.io/provisioner-secret-namespace: hpe-csi-driver
  description: "Volume created using Peer Persistence with the HPE CSI Driver for Kubernetes"
  accessProtocol: iscsi

# Primera customizations
  cpg: SSD_r6
  remoteCopyGroup: new-rcg
  replicationDevices: replication-crd
  provisioning_type: tpvv
  allowOverrides: description,provisioning_type,cpg,remoteCopyGroup
```

Create PersistentVolumeClaim

Next create a `PersistentVolumeClaim` (PVC) based upon the `replicated-storageclass`.

```markdown
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: replicated-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
  storageClassName: replicated-storageclass
```

As the volumes are created using the Remote Copy, the replication between the **default** and **secondary** Primeras will be transparent to Kubernetes. There are [multiple videos](https://youtu.be/Eet92dOra24) out on YouTube demonstrating how automatic transparent failover works with various workloads and within Kubernetes it is no different. In the case of an array failure, automatic transparent failover will manage the pathing between the Primary and Secondary arrays on the Worker nodes so the application IO is not interrupted.

```markdown
kubectl get pvc
NAME             STATUS    VOLUME                            CAPACITY   ACCESS MODES   STORAGECLASS               AGE
replicated-pvc   Bound     pvc-ca03a916-a6fb-434c-bc00-6b8   200Gi      RWO            rep-sc                     1m
```

Finally, run **showrcopy** on both Primeras to see the sync status of the Remote Copy Group.

```markdown
$ showrcopy

Remote Copy System Information
Status: Started, Normal

Target Information

Name              ID Type Status Options Policy
virt-primera-c670  4 IP   ready  -       mirror_config

Link Information

Target            Node  Address     Status Options
virt-primera-c670 0:3:1 172.17.20.5 Up     -
virt-primera-c670 1:3:1 172.17.20.6 Up     -
receive           0:3:1 receive     Up     -
receive           1:3:1 receive     Up     -

Group Information

Name         Target            Status   Role       Mode     Options
new-rcg      virt-primera-c670 Started  Primary    Sync     auto_failover,path_management
  LocalVV                         ID  RemoteVV                          ID SyncStatus    LastSyncTime
  pvc-ca03a916-a6fb-434c-bc00-6b8 168 pvc-ca03a916-a6fb-434c-bc00-6b8   83 Synced        NA
```

This verifies the replication status of the volumes created within your Kubernetes cluster.

By adding Remote Copy support within the HPE CSI Driver along with the ability to perform volume snapshots and cloning capabilities, HPE gives many options within your Disaster Recovery strategy and gives you the peace of mind that your mission-critical application data is protected.

# Next Steps
Stay tuned to the [HPE DEV blog](https://developer.hpe.com/blog) for future posts regarding the HPE CSI Driver for Kubernetes. In the meantime, check out the blog about the new [Volume Mutator capabilities of the HPE CSI Driver](https://developer.hpe.com/blog/8nlLVWP1RKFROlvZJDo9/introducing-kubernetes-csi-sidecar-containers-from-hpe). Also, if you want to learn more about Kubernetes, CSI, and the integration with HPE storage products, you can find a ton of resources out on [SCOD](https://scod.hpedev.io)! If you are already on Slack or an HPE employee, connect with us on Slack. If you are a new user, signup at [slack.hpedev.io](https://slack.hpedev.io). We hang out in #kubernetes, #nimblestorage and #3par-primera.