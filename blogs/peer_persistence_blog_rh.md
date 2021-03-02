# Using HPE Remote Copy Peer Persistence with Red Hat OpenShift and the HPE CSI Operator for Kubernetes

For nearly two decades, HPE and Red Hat have had a long standing partnership providing jointly supported software, platforms and services including many reference architectures and configurations which support HPE and Red Hat customers in their container initiatives.  Whether you’re building physical, virtual, or cloud environments, you can be confident in our tested solutions and world-class services and support. 

The addition of new capabilities the HPE CSI Operator for Kubernetes available in the [Red Hat Ecosystem Catalog](https://catalog.redhat.com/software/operators/detail/5e9874643f398525a0ceb004) and deployed from the [OpenShift OperatorHub](https://operatorhub.io/operator/hpe-csi-operator) never stops. The HPE CSI Operator for Kubernetes has introduced support for HPE Remote Copy Peer Persistence technology for HPE Primera storage systems to protect the data of your mission critical applications running within the Red Hat OpenShift Container platform and other containerized environments. These features come as part of the upstream HPE CSI Driver for Kubernetes that has been developed by HPE and certified by Red Hat. 

## Technology Overview

The HPE CSI Driver for Kubernetes is a comprehensive CSI compliant driver per the Kubernetes CSI specification that provides important data management capabilities for containerized workloads including but not limited to CSI native dynamic provisioning, snapshots, and cloning. Along with the native CSI capabilites, the HPE CSI Driver enables HPE storage specific features to be defined within the StorageClass, such as configuring performance policies or protection templates for persistent volumes on Nimble Storage and enabling HPE Remote Copy Peer Persistence for workloads running on HPE Primera. 

![](https://github.com/c-snell/docs/blob/main/blogs/img/csi_framework.png)

To learn more about the full capabilities of the HPE CSI Driver for Kubernetes check out https://scod.hpedev.io.

Also available are joint Red Hat and HPE Reference Architectures to help customers understand how to build a certified stack using HPE DL/Synergy Servers and HPE storage with OpenShift.

  - [HPE Reference Architecture for Red Hat OpenShift on HPE ProLiant DL380 Gen10 and HPE ProLiant DL360 Gen10 Servers](https://h20195.www2.hpe.com/V2/GetDocument.aspx?docname=a50002456enw)
  - [HPE Reference Architecture for Red Hat OpenShift Container Platform 4 on HPE Synergy and HPE Storage systems](https://h20195.www2.hpe.com/V2/GetDocument.aspx?docname=a50002403enw)

## Feature overview - HPE Remote Copy Peer Persistence for Primera 

As more and more applications migrate into Kubernetes, it is important to ensure that mission-critical applications using persistent storage are highly available and resistant to failure. The HPE Remote Copy Peer Persistence capability within the HPE CSI Operator coupled with Red Hat's enterprise-grade OpenShift Container platform provides enhanced availability for your data and transparent failover between sites in the event of a disaster.  HPE Primera and 3PAR Remote Copy used in conjunction with Red Hat partners like Kasten IO or Commvault for cluster and application state backup and recovery can serve as the foundation for you disaster recovery strategy for your modern applications.

 #img src="https://github.com/c-snell/docs/blob/main/blogs/img/peer_persistence_diagram.png" width="860" height="625">
![](https://github.com/c-snell/docs/blob/main/blogs/img/peer_persistence_diagram_scaled.png)

For information on creating a Remote Copy Peer Persistence configuration, review the [HPE Primera Peer Persistence Host OS Support Matrix](https://techhub.hpe.com/eginfolib/storage/docs/Primera/RemoteCopy/RCconfig/GUID-1F726F48-A372-4ED8-B1D7-9545D091AE98.html#GUID-1F726F48-A372-4ED8-B1D7-9545D091AE98) for the supported host OSs and host persona requirements. Refer to [HPE Primera OS: Configuring data replication using Remote Copy over IP](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=emr_na-a00088914en_us) for more information.

## Get Started

The following guide is based upon the video [Configuring HPE Primera Peer Persistence with HPE CSI Operator for Kubernetes on Red Hat OpenShift](https://www.youtube.com/watch?v=1b9OuadpBfA).

```
embed YouTube video here
```

This video goes through many of the steps shown below to configure HPE Remote Copy Peer Persistence with the HPE CSI Operator as well as demonstrates an array failure and how a deployed workload reacts within a Red Hat OpenShift cluster.

#### Pre-requisites:

  - Single zone Kubernetes cluster
  - All zoning and Remote Copy links configured (RCIP or RCFC) between sites along with Quorum Witness
  - HPE CSI Operator for Kubernetes deployed

#### Deploy the HPE CSI Operator for Kubernetes
Here is a guide along with a tutorial video for deploying the HPE CSI Operator for Kubernetes within a Red Hat OpenShift cluster.

 - [HPE CSI Operator for Kubernetes deployment on Red Hat Openshift](https://scod.hpedev.io/partners/redhat_openshift/index.html)
 - [Video tutorial: Install the HPE CSI Operator for Kubernetes on Red Hat OpenShift](https://www.youtube.com/watch?v=tBjjGuuOn7Q)

#### Create Secret for Remote Copy links
Once the HPE CSI Operatore is deployed, start by creating two `Secrets`. Configure a `Secret` for the HPE Primera array located at each site (i.e. **default-primera-secret** and **secondary-primera-secret**) that were configured as part of the Remote Copy links. 

##### Primary Array

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: default-primera-secret
  namespace: hpe-csi-driver
stringData:
  serviceName: primera3par-csp-svc
  servicePort: "8080"
  backend: 10.10.0.2
  username: admin_user
  password: super_secret_password
```  
**Note:** Verify that the `Namespace` defined within the `Secrets` is the same as the OpenShift `Project` name used when deploying the HPE CSI Operator.

##### Secondary Array

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secondary-primera-secret
  namespace: hpe-csi-driver
stringData:
  serviceName: primera3par-csp-svc
  servicePort: "8080"
  backend: 10.10.0.3
  username: admin_user
  password: super_secret_password
```  

#### Create Peer Persistence CustomResourceDefinition

The next step would be to create a `CustomResourceDefinition` or `CRD` to hold the **target array** information that will be used when creating the replicated volume pairs. 

```yaml
apiVersion: storage.hpe.com/v1
kind: HPEReplicationDeviceInfo
metadata:
  name: replication-crd
spec:
  target_array_details:
  - targetCpg: SSD_r6
    targetName: primera-c670
    targetSecret: secondary-primera-secret
    targetSecretNamespace: hpe-csi-driver
```

#### Create Peer Persistence StorageClass

With the `Secrets` and `CustomResourceDefinition` available the HPE CSI Driver is now configured and ready to provision replicated volumes. 

To get started provisioning replicated volumes, create a replication enabled `StorageClass`. Specify the CSI sidecars to use the `Secret` for the default array, define the `remoteCopyGroup` and the `replicationDevices` parameters in order to enable replication on volumes using this `StorageClass` as well as any additional storage parameters as needed. The HPE CSI Driver can use an existing Remote Copy Group or can create a new one based upon the name specified in the `StorageClass`. During the provisioning process, the volume will be initially created on the primary site array and then the CSI Driver will use the information from the `CRD` to create the replicated volume on the target site Primera array.

**Note:** A Remote Copy group is a group of one or more volumes on an HPE Primera array to be replicated to another system. Because the volumes in a Remote Copy group are related, Remote Copy ensures that the data on the volumes within the group maintain write consistency.

For a full list of available `StorageClass` parameters, see [StorageClass Parameters](https://scod.hpedev.io/container_storage_provider/hpe_3par_primera/index.html#storageclass_parameters).

```yaml
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
  cpg: SSD_r6
  remoteCopyGroup: new-rcg
  replicationDevices: replication-crd
  provisioning_type: tpvv
  allowOverrides: description,provisioning_type,cpg,remoteCopyGroup
```

#### Create PersistentVolumeClaim

Next create a `PersistentVolumeClaim` (PVC) based upon the `replicated-storageclass`.

```yaml
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

Next verify the volume has been created successfully and `Bound` to the cluster

```bash
$ kubectl get pvc
NAME             STATUS    VOLUME                            CAPACITY   ACCESS MODES   STORAGECLASS               AGE
replicated-pvc   Bound     pvc-ca03a916-a6fb-434c-bc00-6b8   200Gi      RWO            rep-sc                     1m
```

As the volumes are dynamically created by the HPE CSI Driver and made available to the OpenShift cluster, replication between the **default** and **secondary** Primera storage arrays using Remote Copy is transparent to Kubernetes. 

Verify the replication status by logging into both Primera arrays to see the sync status of the Remote Copy Group by running `showrcopy` to verify the replication status.

```markdown
$ showrcopy

Remote Copy System Information
Status: Started, Normal

Target Information

Name          ID Type Status Options Policy
primera-c670   4 IP   ready  -       mirror_config

Link Information

Target       Node  Address   Status Options
primera-c670 0:3:1 10.10.0.3 Up     -
primera-c670 1:3:1 10.10.0.3 Up     -
receive      0:3:1 receive   Up     -
receive      1:3:1 receive   Up     -

Group Information

Name         Target            Status   Role       Mode     Options
new-rcg      primera-c670      Started  Primary    Sync     auto_failover,path_management
  LocalVV                         ID  RemoteVV                          ID SyncStatus    LastSyncTime
  pvc-ca03a916-a6fb-434c-bc00-6b8 168 pvc-ca03a916-a6fb-434c-bc00-6b8   83 Synced        NA
```

This verifies that volumes have been created on the primary and remote sites and are synchronized within your Kubernetes cluster. 

In the case of complete array failure, Remote Copy will protect your mission critical applications and minimize the potential for data loss and downtime. Check out the [Peer Persistence video](#get-started) mentioned above to see a demo of what happens to a containerized workload running within OpenShift, when the HPE Remote Copy Quorum Witness detects an array failure and triggers the automatic transparent failover and transitions the workload IO to the secondary site without an outage.

# Learn More

Check out the HPE CSI Operator [in the Red Hat Container Catalog](https://catalog.redhat.com/software/container-stacks/search?p=1&vendor_name=HPE%20Storage&name=HPE%20CSI%20Driver%20for%20Kubernetes). Current supported platforms are HPE Nimble Storage, HPE Primera and HPE 3PAR. Sign up to the HPE DEV Slack community at [slack.hpedev.io](https://slack.hpedev.io) (or login at [hpedev.slack.com](https://hpedev.slack.com) if you already have signed up) to chat with the HPE staff, partners and customers. Also stay informed via our announcements and updates, we hang out in #kubernetes, #nimblestorage and #3par-primera.

- Learn how to [install the HPE CSI Operator](https://scod.hpedev.io/partners/redhat_openshift/index.html) on HPE Storage - Container Orchestrator Documentation (SCOD) portal
- Learn more about HPE Nimble Storage, HPE Primera and HPE 3PAR at [hpe.com/storage](https://hpe.com/storage)
- Join [the HPE Developer Community](https://hpedev.io) to learn from other HPE developers who use HPE products