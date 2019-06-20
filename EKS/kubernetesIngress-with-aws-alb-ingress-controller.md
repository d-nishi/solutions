[Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) is an api object that allows you manage external (or) internal HTTP[s] access to [Kubernetes services](https://kubernetes.io/docs/concepts/services-networking/service/) running in a cluster. [Amazon Elastic Load Balancing Application Load Balancer](https://aws.amazon.com/elasticloadbalancing/features/#Details_for_Elastic_Load_Balancing_Products) (ALB) is a popular AWS service that load balances incoming traffic at the application layer across multiple targets, such as Amazon EC2 instances, in a region. ALB supports multiple features including host or path based routing, TLS (Transport layer security) termination, WebSockets, HTTP/2, AWS WAF (web application firewall) integration, integrated access logs, and health checks.

The [AWS ALB Ingress controller](https://github.com/kubernetes-sigs/aws-alb-ingress-controller) is a controller that triggers the creation of an ALB and the necessary supporting AWS resources whenever a Kubernetes user declares an Ingress resource on the cluster. [TargetGroups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) are created for each backend specified in the Ingress resource. [Listeners](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html) are created for every port specified as Ingress resource annotation. When no port is specified, sensible defaults (80 or 443) are used. [Rules](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html) are created for each path specified in your ingress resource. This ensures that traffic to a specific path is routed to the correct TargetGroup.

In this blog, we will work through a simple example of running ALB based Kubernetes Ingresses with Pulumi EKS, AWS, and AWSX packages.

# Prerequisites to work with Pulumi:
[Install pulumi CLI](https://pulumi.io/quickstart/install.html?__hstc=194006706.92c420b2463a950f50b989da5e9a9de1.1559843842775.1560878104539.1560906741042.31&__hssc=194006706.2.1560906741042&__hsfp=3773980820) and set up your [AWS credentials](https://pulumi.io/quickstart/aws/setup.html?__hstc=194006706.92c420b2463a950f50b989da5e9a9de1.1559843842775.1560878104539.1560906741042.31&__hssc=194006706.2.1560906741042&__hsfp=3773980820). Initialize a new [Pulumi project](https://pulumi.io/reference/project.html?__hstc=194006706.92c420b2463a950f50b989da5e9a9de1.1559843842775.1560878104539.1560906741042.31&__hssc=194006706.2.1560906741042&__hsfp=3773980820) from available templates. We use aws-typescript template here to install all dependencies and save the configuration.

```bash
$ brew install pulumi # download pulumi CLI
$ mkdir eks-alb-ingress && cd eks-alb-ingress
$ pulumi new aws-typescript
$ ls -la
drwxr-xr-x   10 nishidavidson  staff    320 Jun 18 18:22 .
drwxr-xr-x+ 102 nishidavidson  staff   3264 Jun 18 18:13 ..
-rw-------    1 nishidavidson  staff     21 Jun 18 18:22 .gitignore
-rw-r--r--    1 nishidavidson  staff     32 Jun 18 18:22 Pulumi.dev.yaml
-rw-------    1 nishidavidson  staff     91 Jun 18 18:22 Pulumi.yaml
-rw-------    1 nishidavidson  staff    273 Jun 18 18:22 index.ts
drwxr-xr-x   95 nishidavidson  staff   3040 Jun 18 18:22 node_modules
-rw-r--r--    1 nishidavidson  staff  50650 Jun 18 18:22 package-lock.json
-rw-------    1 nishidavidson  staff    228 Jun 18 18:22 package.json
-rw-------    1 nishidavidson  staff    522 Jun 18 18:22 tsconfig.json
```

## STEP 1: Create an EKS cluster and declare ALB Ingress Controller Helm Chart

Update `index.ts` file as follows and run `pulumi up`:

```typescript
import * as awsx from "@pulumi/awsx";
import * as eks from "@pulumi/eks";
import * as k8s from "@pulumi/kubernetes";

const vpc = new awsx.ec2.Vpc("vpc", { subnets: [{ type: "public" }] });
const cluster = new eks.Cluster("eks-cluster", {
 vpcId             : vpc.id,
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

export const clusterName = cluster.eksCluster.name;
export const kubeconfig = cluster.kubeconfig;
```

## STEP 2: Deploy ALB Ingress Controller

Update `index.ts` file from Step 1 and run `pulumi up`:

```typescript
//* STEP 2: Declare ALB Ingress Controller from a Helm Chart
const albingresscntlr = new k8s.helm.v2.Chart("albingresscontroller", {
   chart: "http://storage.googleapis.com/kubernetes-charts-incubator/aws-alb-ingress-controller-0.1.9.tgz",
   values: {
       clusterName: "clusterName",
       autoDiscoverAwsRegion: "true",
       autoDiscoverAwsVpcID: "true",
       name: "alb-ingress-cntroller-0.1.9",
       namespace: "kube-system",
   },
}, { provider: cluster.provider });
```

Confirm the alb-ingress-controller was created as follows:

```bash
$ kubectl get pods -n default | egrep -o alb-ingress[a-zA-Z0-9-]+
alb-ingress-controller-58f44d4bb8lxs6w

$ kubectl logs alb-ingress-controller-58f44d4bb8lxs6w
-------------------------------------------------------------------------------
AWS ALB Ingress controller
  Release:    v1.1.2
  Build:      git-cc1c5971
  Repository: https://github.com/kubernetes-sigs/aws-alb-ingress-controller.git
-------------------------------------------------------------------------------
```

## STEP 3: Deploy Sample Application

Update `index.ts` file from Step 1 and run `pulumi up`:

```typescript
function createNewNamespace(name: string): k8s.core.v1.Namespace {
   //Create new namespace
   return new k8s.core.v1.Namespace(name, { metadata: { name: name } }, { provider: cluster.provider });
 }

//declare 2048 namespace, deployment and service
const nsgame = createNewNamespace("2048-game");

const deploymentgame = new k8s.extensions.v1beta1.Deployment("deployment-game", {
   metadata: { name: "deployment-game", namespace: "2048-game" },
   spec: {
       replicas: 5,
       template: {
           metadata: { labels: { app: "2048" } },
           spec: { containers: [{
                       image: "alexwhen/docker-2048",
                       imagePullPolicy: "Always",
                       name: "2048",
                       ports: [{ containerPort: 80 }]
                   }],
           },
       },
   },
}, { provider: cluster.provider });

const servicegame = new k8s.core.v1.Service("service-game", {
   metadata: { name: "service-2048", namespace: "2048-game" },
   spec: {
       ports: [{ port: 80, targetPort: 80, protocol: "TCP" }],
       type: "NodePort",
       selector: { app: "2048" },
   },
}, { provider: cluster.provider });

//declare 2048 ingress
const ingressgame = new k8s.extensions.v1beta1.Ingress("ingress-game", {
   metadata: {
       name: "2048-ingress",
       namespace: "2048-game",
       annotations: {
           "kubernetes.io/ingress.class": "alb",
           "alb.ingress.kubernetes.io/scheme": "internet-facing"
       },
       labels: { app: "2048-ingress" },
   },
   spec: {
       rules: [{
           http: {
               paths: [{ path: "/*", backend: { serviceName: "service-2048", servicePort: 80 } }]
           }
       }],
   },
}, { provider: cluster.provider });
```
After few seconds, verify the Ingress resource as follows:

```bash
$ kubectl get ingress/2048-ingress -n 2048-game
NAME         HOSTS         ADDRESS         PORTS   AGE
2048-ingress   *    DNS-Name-Of-Your-ALB    80     3m
```
Open a browser. Copy and paste your “DNS-Name-Of-Your-ALB”. You should be to access your newly deployed 2048 game – have fun!
