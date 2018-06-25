AWS Elastic Kubernetes Service (EKS) QuickStart  
===============================================

This solution shows how to create an AWS EKS Cluster and deploy a simple web application with an external Load Balancer. This readme updates an article "Getting Started with Amazon EKS" referenced below and provides a more basic step by step process.

First we'll build an EC2 Instance and configure it to run kubectl for managing the Kubernetes Cluster.  Will then configure an IAM Role Kubernetes can assume to create AWS Resources such as an Elastic Load Balancer.  Will also be using AWS Cloud Formation to create the cluster VPC which will create subnets across 3 AWS Availability Zones (AZ). 

To make this first cluster easy to deploy we'll use a docker image located in DockerHub at kskalvar/web.  This image is nothing more than a simple webapp that returns the current ip address of the container it's running in.       


## Create your Amazon EKS Service Role

Use the AWS Console to configure the EKS IAM Role.  This is a step by step process.

### AWS IAM Dashboard
Select Roles  

Click on "Create role"  
Select "AWS Service"  
Choose the service that will use this role  
```
EKS
```  
Click on "Next: Permissions"  
Click on "Next: Review"  
Enter "Role Name"
```
eks-role
```
Click on "Create role"


## Create your Amazon EKS Cluster VPC

Use the AWS CloudFormation to configure the Cluster VPC.  This is a step by step process.

### AWS CloudFormation Console
Click on "Create Stack"  
Select "Specify an Amazon S3 template URL"  
```
https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/amazon-eks-vpc-sample.yaml
```
Click on "Next"  
```
Stack Name: eks-vpc
```
Click on "Next"  
Click on "Next"
Click on "Create"

Wait for Status CREATE_COMPLETE before proceeding 


## Create your Amazon EKS Cluster

Use the AWS Console to configure the EKS Cluster.  This is a step by step process.

### AWS EKS Dashboard
Click on "Clusters"  

Click on "Create cluster"  
```
Cluster name: eks-cluster
Kubernetes version:  1.10
Role ARN: eks-role
VPC: eks-vpc-VPC
Subnets:  Should preselect all available
Security groups: eks-vpc-ControlPlaneSecurityGroup-*
```

Click on "Create"  

Wait for Status ACTIVE before proceeding

## Configure AWS EC2 Instance

Use the AWS Console to configure the EC2 Instance for kubectl.  This is a step by step process.

### AWS EC2 Dashboard

Click on "Launch Instance"

Choose AMI
```
Amazon Linux AMI 2018.03.0 (HVM), SSD Volume Type (ami-14c5486b)
```  
Click on "Select"

Choose Instance Type
```
t2.micro
```
Click on "Next"

Configure Instance  
Click on "Advanced Details
```
User data
Select "As file"
Click on "Choose File" and Select "cloud-init" from project cloud-deployment directory 
```  
Next

Add Storage  
Next

Add Tags  
Next

Configure Security Group  
Select "Select an existing security group"  
Select "default"

Review and Launch  
Click on "Launch"


## Configure kubectl on EC2 Instance

You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.

### Check to insure cloud-init has completed

See contents of "/tmp/install-eks-support" it should say "installation complete".


### Configure awscli
aws configure
```
AWS Access Key ID []:
AWS Secret Access Key []:
Default region name [None]: us-east-1
```

### Create kubectl configuration file

aws eks list-clusters                                                                  # copy cluster name  
aws eks describe-cluster --name eks-cluster --query cluster.endpoint                   # copy endpoint  
aws eks describe-cluster --name eks-cluster  --query cluster.certificateAuthority.data # copy certificate  

git clone https://github.com/kskalvar/aws-eks-cluster-quickstart.git

mkdir -p ~/.kube  
cp ~/aws-eks-cluster-quickstart/kube-config/control-kubeconfig ~/.kube  
cd ~/.kube  
edit config-eks-cluster # replacing <cluster-name>, <endpoint-url>, and <base64-encoded-ca-cert> with information above  

export KUBECONFIG=~/.kube/control-kubeconfig  

kubectl get svc  # test 


## Launch and Configure Amazon EKS Worker Nodes

Use the AWS CloudFormation to configure the Worker Nodes.  This is a step by step process.

### AWS CloudFormation Console
Click on "Create Stack"  
Select "Specify an Amazon S3 template URL"  
```
https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/amazon-eks-nodegroup.yaml
```
Click on "Next"  
```
Stack Name: eks-worker-nodes
ClusterNamme: eks-cluster
ClusterControlPlaneSecurityGroup: eks-cluster-*-ControlPlaneSecurityGroup-*
NodeGroupName: eks-cluster-nodes
NodeImageId: ami-dea4d5a1
KeyName: <Your AWS KeyName>
VpcId: eks-cluster
Subnets: Subnet01, Subnet02, Subnet03
```
Click on "Next"  
Click on "Next"
Select Check Box "I acknowledge that AWS CloudFormation might create IAM resources"
Click on "Create"

Wait for Status CREATE_COMPLETE before proceeding  
You should be able to see the additional nodes visible in AWS EC2 Console

## Enable Worker Nodes to Join Your Cluster

curl -O https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/aws-auth-cm.yaml

edit aws-auth-cm.yaml # replacing "<ARN of instance role (not instance profile)>" with NodeInstanceRole from output of CloudFormation script "eks-worker-nodes" 
kubectl apply -f aws-auth-cm.yaml

kubectl get nodes --watch

You should be able to see several nodes appear in "STATUS Ready" shortly


## Deploy WebApp to Your Cluster

### create pod
kubectl run web --image=kskalvar/web --port=5000

#### scale pod
kubectl scale deployment web --replicas=3

#### show pods running
kubectl get pods --output wide

### create load balancer
kubectl expose deployment web --port=80 --target-port=5000 --type="LoadBalancer"

### get aws external load balancer external address
kubectl get service web --output wide

### test from browser

### kill application
kubectl delete deployment,service web


## Remove AWS EKS Cluster

### AWS CloudFormation Delete eks-worker-nodes Stack

### AWS EKS Delete eks-cluster
Wait for cluster to be deleted before proceeding

### AWS CloudFormation Delete eks-cluster-vpc

### AWS EC2 Delete kubectl Instance




## References
Getting Started with Amazon EKS  
https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html
