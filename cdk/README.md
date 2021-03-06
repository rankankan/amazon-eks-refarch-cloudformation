# Amazon EKS cluter with AWS CDK

This sample CDK scripts help you provision your Amaozn EKS cluster, a default nodegroup and kubernetes resources.



## Setup

```bash
# install the nvm installer
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
# nvm install 
nvm install lts/dubnium
nvm alias default lts/dubnium
# install AWS CDK
npm i -g aws-cdk
# check cdk version
cdk --version
```



OK. Let's deploy our Amazon EKS environment.



## Deploy

```bash
# install other required npm modules
npm i
# build the .ts to .js with tsc
npm run build
# cdk bootstrapping (only for the 1st time)
cdk bootstrap
# cdk deploy
cdk deploy EksStack -c region=us-west-2
```

Outputs

```bash
Outputs:
CdkEksStack.ClusterConfigCommand43AAE40F = aws eks update-kubeconfig --name cdk-eks --role-arn arn:aws:iam::112233445566:role/CdkEksStack-AdminRole38563C57-14ON0YFUELKGC
```



## Generate kubeconfig

Just copy and paste the Outputs and execute it to update your kubeconfig

```bash
aws eks update-kubeconfig --name cdk-eks --role-arn arn:aws:iam::112233445566:role/CdkEksStack-AdminRole38563C57-14ON0YFUELKGC
```

OK. Now you got it!

```bash
kubectl get no
NAME                                          STATUS   ROLES    AGE   VERSION
ip-172-31-11-179.us-west-2.compute.internal   Ready    <none>   40s   v1.14.6-eks-5047ed
ip-172-31-14-197.us-west-2.compute.internal   Ready    <none>   59s   v1.14.6-eks-5047ed
```



DONE!

This sample gives you:

1) A dedicated VPC with default configuration

2) Amazon EKS cluster in the VPC

3) A default nodegroup of **2x m5.large** instances

4) Self-configured `aws-auth` ConfigMap



## Create K8s Resources in CDK

You may also create K8s resourdces alone with the Amazon EKS cluster:

```js
    const appLabel = { app: "hello-kubernetes" };

    const deployment = {
      apiVersion: "apps/v1",
      kind: "Deployment",
      metadata: { name: "hello-kubernetes" },
      spec: {
        replicas: 3,
        selector: { matchLabels: appLabel },
        template: {
          metadata: { labels: appLabel },
          spec: {
            containers: [
              {
                name: "hello-kubernetes",
                image: "paulbouwer/hello-kubernetes:1.5",
                ports: [{ containerPort: 8080 }]
              }
            ]
          }
        }
      }
    };

    const service = {
      apiVersion: "v1",
      kind: "Service",
      metadata: { name: "hello-kubernetes" },
      spec: {
        type: "LoadBalancer",
        ports: [{ port: 80, targetPort: 8080 }],
        selector: appLabel
      }
    };

    cluster.addResource('hello-kub', service, deployment);
```

You will get a k8s delployment and service immediately after you deploy the CDK scripts.



![](https://pbs.twimg.com/media/EB8IMZ-U8AA85Cl?format=jpg&name=4096x4096)

![](https://pbs.twimg.com/media/EB8IMZ-UwAAuW1u?format=jpg&name=4096x4096)





## Destroy the stack

```bash
# destroy the stack
cdk destroy EksStack -c region=us-west-2
```


## Using the default VPC

If you do not specify the vpc, `aws-eks` CDK construct lib will create a default VPC for you. If you intend to use the default VPC.

```js

    const vpc = Vpc.fromLookup(this, 'VPC', {
      isDefault: true
    })

    // first define the role
    const clusterAdmin = new iam.Role(this, 'AdminRole', {
      assumedBy: new iam.AccountRootPrincipal()
    });

    // eks cluster with nodegroup of 2x m5.large instances in dedicated vpc with default configuratrion
    const cluster = new eks.Cluster(this, 'Cluster', {
      vpc,
      clusterName: 'cdk-eks',
      mastersRole: clusterAdmin
    });    

```
Make sure:
1. All your private subnets have a default routing table with NAT gateway for `0.0.0.0/0`
2. All your public subnets have enable the `auto-assign public IP` option
3. Tag **kubernetes.io/role/internal-elb=1** on all your private subnets in your default VPC


## Specify SSH KeyPair for your default capacity

You can't specify SSH KeyPair for your default capacity with `eks.Cluster()` at this moment. A quick hack is to use `addPropertyOverride` like this:

```js
    const cluster = new eks.Cluster(this, 'Cluster', {
      vpc,
      clusterName: 'cdk-eks',
      mastersRole: clusterAdmin
    });  

    // specify ssh keypair for your default capacity
    const resourceLaunchConfig = cluster.defaultCapacity!.node.findChild('LaunchConfig') as CfnLaunchConfiguration
    resourceLaunchConfig.addPropertyOverride('KeyName', 'your-keypair-name')
```
