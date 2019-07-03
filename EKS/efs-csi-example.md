![alt text](efs-csi-drive-amazon-eks-pulumi-featuredimg.png)
# Using EFS CSI Driver on Amazon EKS with Pulumi Crosswalk for AWS

The Amazon Elastic File System Container Storage Interface (CSI) Driver implements the [CSI specification](https://github.com/container-storage-interface/spec/blob/master/spec.md) for container orchestrators to manage the lifecycle of Amazon EFS filesystems.
In this blog, we will work through an example that shows how to use AWS EFS with Amazon EKS worker nodes using Pulumi libraries (EKS, AWSX, AWS) and Pulumi Service.

## Prerequisites

* An account on [https://app.pulumi.com](https://app.pulumi.com/) with an organization. 
* The latest `pulumi` CLI. Installation instructions are [here](https://pulumi.io/quickstart/install.html).
* A bare repository. Set the remote URL to be your GitLab project.

We will work with two Pulumi stacks in this example, one for the Amazon EKS cluster and AWS EFS CSI components caled k8sinfra. The other for the application and its storage class, persistent volume and persistent volume claim called app. The AWS EFS CSI (Container Storage Interface) is based on the initial [AWS EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver/) work done by Kubernetes [SIG-AWS](https://github.com/kubernetes/community/tree/master/sig-aws).

## Create Amazon EKS cluster and AWS EFS CSI components as part of Pulumi Stack Infra:

```bash
$ mkdir k8sinfra && cd k8sinfra
$ pulumi new typescript
$ npm install --save @pulumi/eks @pulumi/kubernetes
```

The code below can be pasted in `index.ts` to create a default EKS cluster with two subnets in the new VPC. We then declare EFS endpoint and mount the same to both subnets so the the EKS worker nodes can access them as and when needed. 

```typescript
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
import * as eks from "@pulumi/eks";
import * as k8s from "@pulumi/kubernetes";

//* STEP 1: Create an EKS Cluster

const vpc = new awsx.Network("vpc", { usePrivateSubnets: false });
export const cluster = new eks.Cluster("eks-cluster", {
  vpcId             : vpc.vpcId,
  subnetIds         : vpc.publicSubnetIds,
  instanceType      : "t2.medium",
  version           : "1.12",
  nodeRootVolumeSize: 200,
  desiredCapacity   : 3,
  maxSize           : 4,
  minSize           : 3,
  deployDashboard   : false,
  vpcCniOptions     : {
    warmIpTarget    : 4,
  },
});

const nodesg = cluster.nodeSecurityGroup;
const nodesubnetId = cluster.core.subnetIds;
export const kubeconfig = cluster.kubeconfig.apply(JSON.stringify);

//* STEP 2: Create an EFS endpoint. Instances connect to a file system by using mount targets you create. 
// Creating a mount target in each of the EKS cluster VPC's subnet allows EC2 instances in the VPC to access the file system.

export const efsFilesystem = new aws.efs.FileSystem("MyEFS", {
  tags: { Name: "myEFS" },
});

export const efsFilesystemId = efsFilesystem.id;

new aws.efs.MountTarget("MyEFS-MountTarget1", {
  fileSystemId: efsFilesystemId,
  securityGroups: [ nodesg.id ],
  subnetId: nodesubnetId[0],
});

new aws.efs.MountTarget("MyEFS-MountTarget2", {
  fileSystemId: efsFilesystemId,
  securityGroups: [ nodesg.id ],
  subnetId: nodesubnetId[1],
});
```

## Create the CSI driver components

Currently only static provisioning for AWS EFS CSI drivers is supported by SIG-AWS. Implies, AWS EFS filesystem needs to be created manually on AWS first. After that it can be mounted inside a container as a volume using the driver. The code below will allow you to deploy allow the CSI Driver components on the Amazon EKS cluster.

```typescript
//* STEP 3: Install an EFS CSI Driver node - svc account, clusterrole, clusterrolebinding and daemonset components 

const svcacntcsiNode = new k8s.core.v1.ServiceAccount ("csi-node-sa", {
  metadata: { name: "csi-node-sa", namespace: "kube-system"},
}, { provider: cluster.provider });

const clusterolecsiNode = new k8s.rbac.v1.ClusterRole("csi-node", {
  metadata: { name: "csi-node", namespace: "default"},
  rules: [
    { 
    apiGroups: [""],
    resources: ["secrets"],
    verbs: ["get", "list"],
  },
  { 
    apiGroups: [""],
    resources: ["nodes"],
    verbs: ["get", "list", "update"],
  },
  { 
    apiGroups: [""],
    resources: ["namespaces"],
    verbs: ["get", "list"],
  },  
  { 
    apiGroups: [""],
    resources: ["persistentvolumes"],
    verbs: ["get", "list", "watch", "update"],
  },
  { 
    apiGroups: ["storage.k8s.io"],
    resources: ["volumeattachments"],
    verbs: ["get", "list", "watch", "update"],
  },  
  { 
    apiGroups: ["csi.storage.k8s.io"],
    resources: ["csinodeinfos"],
    verbs: ["get", "list", "watch", "update"],
  },
 ],
}, { provider: cluster.provider });

const clusterolebindingcsiNode = new k8s.rbac.v1.ClusterRoleBinding("csi-node", {
  metadata: { name: "csi-node", namespace: "default" },
  subjects: [{ 
    kind: "ServiceAccount",
    name: "csi-node-sa", 
    namespace: "default", 
  }],
  roleRef: { 
    kind: "ClusterRole", 
    name: "csi-node", 
    apiGroup: "rbac.authorization.k8s.io",
  },
}, { provider: cluster.provider });

const daemonsetcsiNode = new k8s.apps.v1beta2.DaemonSet("efs-csi-node", {
  metadata: { name: "efs-csi-node", namespace: "kube-system" },
  spec: {
    selector: { matchLabels: { app: "efs-csi-node" } },
    template: { 
      metadata: { labels: { app: "efs-csi-node" } },
      spec: {
          serviceAccount: "csi-node-sa",
          hostNetwork: true,
          containers: [
            {
                name: "efs-plugin",
                securityContext: { 
                  privileged: true, 
                },
                image: "amazon/aws-efs-csi-driver:latest",
                imagePullPolicy: "Always",
                args: [ "--endpoint=$(CSI_ENDPOINT)", "--logtostderr", "--v=5" ],
                env: [
                  { name: "CSI_ENDPOINT", value: "unix:/csi/csi.sock" }
                ],
                volumeMounts: [
                  { name: "kubelet-dir", mountPath: "/var/lib/kubelet", mountPropagation: "Bidirectional" },
                  { name: "plugin-dir", mountPath: "/csi" },
                  { name: "device-dir", mountPath: "/dev" }
                ],
            },
            {
                name: "csi-driver-registrar",
                image: "quay.io/k8scsi/driver-registrar:v0.4.2",
                imagePullPolicy: "Always",
                args: [ "--csi-address=$(ADDRESS)", "--mode=node-register", "--driver-requires-attachment=true", "--pod-info-mount-version=v1", "--kubelet-registration-path=$(DRIVER_REG_SOCK_PATH)", "--v=5" ],
                env: [
                  { name: "ADDRESS", value: "/csi/csi.sock" }, 
                  { name: "DRIVER_REG_SOCK_PATH", value: "/var/lib/kubelet/plugins/efs.csi.aws.com/csi.sock" }, 
                  { name: "KUBE_NODE_NAME", valueFrom: { fieldRef: { fieldPath : "spec.nodeName" } } },
                ],
                volumeMounts: [
                  { name: "plugin-dir", mountPath: "/csi" },
                  { name: "registration-dir", mountPath: "/registration" },
                ],
          },
          ],
          volumes: [ 
            { name: "kubelet-dir", hostPath: { path: "/var/lib/kubelet", type: "Directory" } },
            { name: "plugin-dir", hostPath: { path: "/var/lib/kubelet/plugins/efs.csi.aws.com/", type: "DirectoryOrCreate" } },
            { name: "registration-dir", hostPath: { path: "/var/lib/kubelet/plugins/", type: "Directory" } },
            { name: "device-dir", hostPath: { path: "/dev", type: "Directory" } }
          ],
      },
    },
  }, 
}, { provider: cluster.provider });

//* STEP 4: Install an EFS CSI Driver controller - svc account, clusterrole, clusterrolebinding, statefulset components 

const svcacctcsiController = new k8s.core.v1.ServiceAccount ("csi-controller-sa", {
  metadata: { name: "csi-controller-sa", namespace: "kube-system"},
}, { provider: cluster.provider });

const clusterrolecsiController = new k8s.rbac.v1.ClusterRole("external-attacher-role", {
  metadata: { name: "external-attacher-role", namespace: "default"},
  rules: [
    { 
    apiGroups: [""],
    resources: ["persistentvolumes"],
    verbs: ["get", "list", "watch", "update"],
  },
  { 
    apiGroups: [""],
    resources: ["nodes"],
    verbs: ["get", "list", "watch"],
  },
  { 
    apiGroups: ["storage.k8s.io"],
    resources: ["volumeattachments"],
    verbs: ["get", "list","watch", "update"],
  },
 ],
}, { provider: cluster.provider });

const clusterrolebindingcsiController = new k8s.rbac.v1.ClusterRoleBinding("csi-attacher-role", {
  metadata: { name: "csi-attacher-role", namespace: "default" },
  subjects: [{ 
    kind: "ServiceAccount",
    name: "csi-controller-sa", 
    namespace: "kube-system", 
  }],
  roleRef: { 
    kind: "ClusterRole", 
    name: "external-attacher-role", 
    apiGroup: "rbac.authorization.k8s.io",
  },
}, { provider: cluster.provider });

const statefulsetcsiController = new k8s.apps.v1beta1.StatefulSet("efs-csi-controller", {
  metadata: { name: "efs-csi-controller", namespace: "kube-system" },
  spec: {
    serviceName: "efs-csi-controller",
    replicas: 1,
    template: { 
      metadata: { labels: { app: "efs-csi-node" } },
      spec: {
        serviceAccount: "csi-controller-sa",
        priorityClassName: "system-cluster-critical",
        tolerations: [{ key: "CriticalAddonsOnly", operator: "Exists" }],
        containers: [
          {
              name: "efs-plugin",
              image: "amazon/aws-efs-csi-driver:latest",
              imagePullPolicy: "Always",
              args: [ "--endpoint=$(CSI_ENDPOINT)", "--logtostderr", "--v=5" ],
              env: [
                { name: "CSI_ENDPOINT", value: "unix:///var/lib/csi/sockets/pluginproxy/csi.sock" }
              ],
              volumeMounts: [
                { name: "socket-dir", mountPath: "/var/lib/csi/sockets/pluginproxy/"},
              ],
          },
          {
              name: "csi-attacher",
              image: "quay.io/k8scsi/csi-attacher:v0.4.2",
              imagePullPolicy: "Always",
              args: [ "--csi-address=$(ADDRESS)", "--v=5" ],
              env: [
                { name: "ADDRESS", value: "/var/lib/csi/sockets/pluginproxy/csi.sock" },
              ],
              volumeMounts: [
                { name: "socket-dir", mountPath: "/var/lib/csi/sockets/pluginproxy/"},
              ],
        },
        ],
        volumes: [ 
          { name: "socket-dir", emptyDir: { } },
        ],
      },
    },
  }, 
}, { provider: cluster.provider });
```

## Deploy a sample app with the EFS volume mounts:

Once the step above is complete, you will be ready to deploy your k8s storage class, persistent volume and persistent volume claim based on the AWS EFS CSI driver as well as the sample k8s application. 

Lets create a new stack for this called k8sapp as follows:

```bash
$ mkdir k8sapp && cd k8sapp
$ pulumi new typescript
$ npm install --save @pulumi/eks @pulumi/kubernetes
```

Let's now declare the storage class, pv, pvc and application pod to complete our k8s application set-up for this example. We will update `index.ts` file again and run `pulumi up`:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

const env = pulumi.getStack();
const cluster = new pulumi.StackReference(`d-nishi/eks-efs-cluster/${env}`);
const kubeconfig = cluster.getOutput("kubeconfig");
const efsFilesystemId = cluster.getOutput("efsFilesystemId");

const k8sProvider = new k8s.Provider("cluster", {
    kubeconfig: kubeconfig,
 });

//* STEP 5: Create a storage class, persistent volume and persistent volume claim

export const storageclassEFS = new k8s.storage.v1.StorageClass("efs-sc", {
    metadata: { name: "efs-sc" },
    provisioner: "efs.csi.aws.com"
  }, { provider: k8sProvider });
  
  const pvEFS = new k8s.core.v1.PersistentVolume("efs-pv", {
  metadata: { name: "efs-pv" },
  spec: { 
    capacity: { storage: "5Gi" },
    volumeMode: "Filesystem",
    accessModes: [ "ReadWriteOnce" ],
    persistentVolumeReclaimPolicy: "Recycle",
    storageClassName: "efs-sc",
    csi: { 
      driver: "efs.csi.aws.com", 
      volumeHandle: efsFilesystemId,
    }
  }
  }, { provider: k8sProvider });
  
  const pvcEFS = new k8s.core.v1.PersistentVolumeClaim("efs-claim", {
  metadata: { name: "efs-claim" },
  spec: { 
    accessModes: [ "ReadWriteOnce" ],
    storageClassName: "efs-sc",
    resources: { requests: { storage: "5Gi" } }
  }
  }, { provider: k8sProvider });

 //* STEP 6: Mount the endpoint to pod in EKS cluster

export const newPod = new k8s.core.v1.Pod("efs-app", {
  metadata: { name: "efs-app" },
  spec: {
    containers: [{ 
      name: "app" , 
      image: "centos", 
      command: ["/bin/sh"],
      args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"],
      volumeMounts: [{ 
        name: "persistent-storage", 
        mountPath: "/data",
      }],
    }],
    volumes: [{
      name: "persistent-storage",
      persistentVolumeClaim: { claimName: "efs-claim" }
    }],
  }
}, { provider: k8sProvider });
```

Verify the pod is running and that data is being written into the EFS filesystem using:
`kubectl exec -ti efs-app -- tail -f /data/out.txt`

This brings us to the end of our solution with Pulumi and AWS EFS on Amazon EKS. For more examples, refer to Pulumi's open source repository [here](https://github.com/pulumi/examples). Refer to my other blogs on Kubernetes [here](https://blog.pulumi.com/author/nishi-davidson).
