# Building Amazon EKS cluter and nodegroup with AWS CDK

This sample CDK scripts help you provision your Amaozn EKS cluster and nodegroup. Two options are provided below:

# Option #1

Build the Amazon EKS with AWS CDK and update **aws-auth** ConfigMap manually.

## Setup

```bash
# install the nvm installer
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
# nvm install 
nvm install lts/dubnium
nvm alias default lts/dubnium
# install AWS CDK
npm i -g aws-cdk
# check cdk version, make sure your version >=1.0.0
cdk --version
1.2.0 (build 6b763b7)
# install other required npm modules
npm install
# build the index.ts to index.js with tsc
npm run build
# cdk bootstrapping
cdk bootstrap
# cdk synth
cdk synth
# cdk deploy
cdk deploy
```


CDK will provision Amazon EKS cluster as well as nodegroup in a single stack and generate outputs like this:

```
Outputs:
EKS-CDK-demo.eksdemocdkClusterNameE757A622 = eksdemo-cdk
EKS-CDK-demo.ClusterARN = arn:aws:eks:ap-northeast-1:903779448426:cluster/eksdemo-cdk
EKS-CDK-demo.eksdemocdkNodesInstanceRoleARN7B26F304 = arn:aws:iam::903779448426:role/EKS-CDK-demo-eksdemocdkNodesInstanceRoleAA83309B-BWBA4TG1XLJP
```

## Generate kubeconfig


```bash
# generate kubeconfig for our cluster
$ aws eks update-kubeconfig --name eksdemo-cdk
```

## Update aws-auth ConfigMap

You need to specify your own NodeInstanceRoleARN above to replace the provided `aws-auth-cm.yaml` and update it with `kubectl apply`.


See [official document](https://docs.aws.amazon.com/en_us/eks/latest/userguide/add-user-role.html) for more details.

```bash
# we specify our NodeInstanceRoleARN generated from CDK, replacing the content of aws-auth-cm.yaml on-the-fly 
# followed by immediate 'kubectl apply -f'
# PLEASE MAKE SURE you specify your own MY_InstanceRole_ARN
MY_InstanceRole_ARN='arn:aws:iam::903779448426:role/EKS-CDK-demo-eksdemocdkNodesInstanceRoleAA83309B-BWBA4TG1XLJP'
curl -sL https://github.com/aws-samples/amazon-eks-refarch-cloudformation/raw/master/files/aws-auth-cm.yaml | \
sed -e "s?<your role arn here>?${MY_InstanceRole_ARN}?g" | kubectl apply -f -                                            
```

Response

```
configmap/aws-auth created
```

And then you'll be able to list the nodes
```
$ kubectl get no
NAME                                             STATUS     ROLES    AGE     VERSION
ip-10-0-17-49.ap-northeast-1.compute.internal    NotReady   <none>   9m50s   v1.13.7-eks-c57ff8
ip-10-0-22-228.ap-northeast-1.compute.internal   NotReady   <none>   9m50s   v1.13.7-eks-c57ff8
```



## Destroy the stack

```bash
# destroy the stack
cdk destroy
```

## 



# Option #2

1. Build the `VPC` and `EKS` stacks seperatedly
2. Using `AmazonEKSAdminRole` IAM role to create the cluster and update `aws-auth` ConfigMap from Lambda as cloudformation custom resource automatically.

## Prerequisities

Follow [Create AmazonEKSAdminRole IAM Role](https://github.com/aws-samples/amazon-eks-refarch-cloudformation/blob/master/README.md#create-amazoneksadminrole-iam-role) to create your **AmazonEKSAdminRole** IAM Role in your AWS account.

## Setup

```bash
# install the nvm installer
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
# nvm install 
nvm install lts/dubnium
nvm alias default lts/dubnium
# install AWS CDK
npm i -g aws-cdk
# check cdk version, make sure your version >=1.0.0
cdk --version
1.2.0 (build 6b763b7)
# install other required npm modules
npm install
# build the index.ts to index.js with tsc
npm run build
# cdk bootstrapping
cdk bootstrap
# cdk synth with -r to specify your AmazonEKSAdminRole IAM role ARN
cdk synth -c region=ap-northeast-1 -c clusterVersion=1.13 -r arn:aws:iam::903779448426:role/AmazonEKSAdminRole -a index-with-role.js
# cdk deploy the EKS-VPC stack first
cdk deploy EKS-VPC \
-c region=ap-northeast-1 \
-r arn:aws:iam::903779448426:role/AmazonEKSAdminRole \
-a index-with-role.js

# cdk deploy the EKS-Main stack
cdk deploy EKS-Main \
-c region=ap-northeast-1 \
-c clusterVersion=1.13 \
-c lambdaRoleArn=arn:aws:iam::903779448426:role/AmazonEKSAdminRole \
-r arn:aws:iam::903779448426:role/AmazonEKSAdminRole \
-a index-with-role.js
```



Behind the scene, 4 cloudformation stacks will be created:

1. **EKS-VPC** is the infrastructure stack for your Amazon EKS cluster.
2. **EKS-Main** is your primary Amazon EKS stack including the cluster and nodegrup.
3. **EKS-Main-lambdaLayerKubectl-{RANDOM_ID}** lambda layer installed from SAR for **EKS-Main-sam**
4. **EKS-Main-sam-{RANDON_ID}** is a nested stack with Lambda function generated from SAR(Serverless App Repository) to help you update `aws-auth-cm` automatically.



## Generate kubeconfig

```bash
# generate kubeconfig for our cluster
$ aws --region ap-northeast-1 eks update-kubeconfig --name eksdemo-cdk --role-arn arn:aws:iam::903779448426:role/AmazonEKSAdminRole
```

Output

```bash
Updated context arn:aws:eks:ap-northeast-1:903779448426:cluster/eksdemo-cdk in /Users/pahud/.kube/config
```

Make sure you specify `—role-arn` as Amazon EKS cluster was just created by this role and we need to assume this role before we can execute the `kubectl` commands.

Let's list the nodes

```bash
$ kubectl get no
NAME                                             STATUS   ROLES    AGE   VERSION
ip-10-0-16-10.ap-northeast-1.compute.internal    Ready    <none>   46m   v1.13.7-eks-c57ff8
ip-10-0-21-170.ap-northeast-1.compute.internal   Ready    <none>   46m   v1.13.7-eks-c57ff8
```



NOTE: You don't have to manually update `aws-auth` ConfigMap in this option - AWS Lamdda from the custom resource will take care of that for you.



## Destroy the stack

```bash
# destroy the stack
cdk destroy EKS* -c region=ap-northeast-1 -a index-with-role.js
```



## FAQ

1. How to specify different `region`?

You can pass `region` context variable like this:

```bash
# cdk synth
cdk synth -c region=us-west-2
# cdk deploy
cdk deploy -c region=us-west-2
```

2. Can I have different NAT gateway in each public subnet?

By default, this sample will generate only one single NAT gateway for you in one of the public subnets for cost saving and avoid reaching the max EIP number limit(default:5).
If you prefer to have different NAT gateway provisioned for each public subnet, just update the `vpc` constructor in `index.ts`.

```ts
        // VPC with public subnet in every AZ and private subnet in every AZ
        // each public subnet will privision individual NAT gateway with EIP attached.
        // This is how CDK provision VPC resource for you by default.
        const vpc = new ec2.Vpc(this, 'cdk-EKS-vpc');
        
        // VPC with only single NAT gateway in one of the public subnets
        // const vpc = new ec2.Vpc(this, 'cdk-EKS-vpc', {
        //   cidr: '10.0.0.0/16',
        //   natGateways: 1,
        //   natGatewaySubnets: {subnetName: 'Public'},
        //   subnetConfiguration: [
        //     {
        //       cidrMask: 22,
        //       name: 'Public',
        //       subnetType: ec2.SubnetType.PUBLIC, 
        //     },
        //     {
        //       cidrMask: 22,
        //       name: 'Private',
        //       subnetType: ec2.SubnetType.PRIVATE, 
        //     },
        //   ],
        // });   
```

3. Can I specify different Amazon EKS cluster version?

Yes.

```bash
# Specify 1.13 as the Amazon EKS cluster version
cdk synth -c clusterVersion=1.13
cdk deploy -c clusterVersion=1.13
```