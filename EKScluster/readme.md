# Deploying EKS the Hard Way

![](/_temp/kubernetes1.jpg)

[EKS](https://aws.amazon.com/eks/) is a managed kubernetes service from Amazon AWS. There is nothing difficult about it(Sorry for the misleading title!). That being said, there are still a lot of moving pieces in the backend, to make this peachy platform work.   
So instead of going the universally acclaimed easiest route of deploying though [EKSCTL](https://eksctl.io/), I will be going into the individual components that we need to make this cluster work, there by understanding more about them instead of just spinning up through a cli command.

#### Assuming you have a fresh AWS account with just a root user to go in, we will do this setup in 3 phases.
### 1. Create a three Subnet highly available Network Stack on AWS. 
[Please read my blog](https://dev.to/anupamncsu/aws-virtual-private-cloud-setup-as-a-city-analogy-3ekc) to understand and create it. It will be just a single Cloudformation stack to get this setup. We need this to deploy our worker nodes of EKS cluster into.
### 2. Create an admin user to deploy and administer the EKS cluster.
It is always a best practice to not use the root account to use the AWS. [In this blog I explain how to create a user](https://dev.to/anupamncsu/aws-access-through-users-groups-3253) with another cloudformation stack. The blog explains to create two users, but we will be using the admin user for this tutorial for simplicity. Also, how to configure the user access from your local desktop. We will need this access after we have the cluster to deploy apps into EKS. The famous kubectl will have this as a prerequisite to work. Going forward with the assumption that you have this step completed and you have a working AWS CLI with a admin profile.

### 3. Deploy the cluster
This section consist of creating the IAM resources for the cluster and then the compute resources. This is the meat and potato of this blog so lets explain them in detail.

-----
#### EKS IAM Resources
The EKs cluster consists of two defined sections, the EKS **service** that AWS maintains and the worker **node** cluster that you are responsible for providing the details and security for. 
- Both these sections have two different IAM roles with a defined sets of Managed policies to attach to them. These can be defined as :
```
###### EKS Service Role ######
  EksClusterServiceRole:
    Type: AWS::IAM::Role
    Properties: 
      AssumeRolePolicyDocument: 
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
              - eks.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Description: EksClusterServiceRole
      ManagedPolicyArns: 
        - arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
        - arn:aws:iam::aws:policy/AmazonEKSServicePolicy
      MaxSessionDuration: 3600
      Path: /
      RoleName: !Ref EksClusterServiceRoleName
      Tags: 
        - Key: Environment
          Value: !Ref Environment

###### EKS Node Role ######
  EksClusterNodeRole:
    Type: AWS::IAM::Role
    Properties: 
      AssumeRolePolicyDocument: 
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
              - ec2.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Description: EksClusterNodeRole
      ManagedPolicyArns: 
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
      MaxSessionDuration: 3600
      Path: /
      RoleName: !Ref EksClusterNodeRoleName
      Tags: 
        - Key: Environment
          Value: !Ref Environment
```
- Both these compute sections need their own security groups. To design these groups we have to think beside accessing each other for the functioning of the cluster(**port 1025-65535**), who accesses these groups. The Service section is being accessed by outside when we run Kubectl from our desktop. We can clear a definite IP, a known CIDR range if we know what that IP might be. In this case for simplicity, I have opened it to the whole world. The worker nodes need to be accesed by other services accessing the apps running inside it. That will again need futher scrutiny based on what accesses your apps. For example, If you plan to deploy web applications and expose them to only a internal load-balancer, you can keep a defined port open to that load balancer and that will make it secure. For the purpose of simplicity, I have kept the https port 443 open to the whole world.
These two groups can be defined as follows:
```
## Service Security Group ##
  EksServiceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties: 
      GroupDescription: EksServiceSecurityGroup
      GroupName: !Ref EksServiceSecurityGroupName
      Tags: 
        - Key: Environment
          Value: !Ref Environment
      VpcId: !Ref Vpc

## Node Security Group ##
  EksNodeSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    DependsOn: EksServiceSecurityGroup
    Properties: 
      GroupDescription: EksNodeSecurityGroup
      GroupName: !Ref EksNodeSecurityGroupName
      Tags: 
        - Key: Environment
          Value: !Ref Environment
      VpcId: !Ref Vpc 

### Service  Security group Ingress ###
  EksServiceSecurityGroupIngress1:
    Type: AWS::EC2::SecurityGroupIngress
    DependsOn: EksNodeSecurityGroup
    Properties:
      GroupId: !Ref EksServiceSecurityGroup
      IpProtocol: tcp
      FromPort: 443
      ToPort: 443
      SourceSecurityGroupId: !Ref EksNodeSecurityGroup

### Service  Security group Egress ###
  EksServiceSecurityGroupEgress1:    
    Type: AWS::EC2::SecurityGroupEgress
    DependsOn: EksNodeSecurityGroup
    Properties: 
      DestinationSecurityGroupId: !Ref EksNodeSecurityGroup
      FromPort: 1025
      ToPort: 65535 
      IpProtocol: tcp
      GroupId: !Ref EksServiceSecurityGroup

  EksServiceSecurityGroupEgress2:    
    Type: AWS::EC2::SecurityGroupEgress
    DependsOn: EksNodeSecurityGroup
    Properties: 
      DestinationSecurityGroupId: !Ref EksNodeSecurityGroup
      FromPort: 443
      ToPort: 443 
      IpProtocol: tcp
      GroupId: !Ref EksServiceSecurityGroup

### Node Security group Ingress ###
  # Open every port to itself
  EksNodeSecurityGroupIngress1:
    Type: AWS::EC2::SecurityGroupIngress
    DependsOn: EksNodeSecurityGroup
    Properties:
      GroupId: !Ref EksNodeSecurityGroup
      IpProtocol: -1
      FromPort: -1
      ToPort: -1
      SourceSecurityGroupId: !Ref EksNodeSecurityGroup

  # open 1024-65535 to service SG
  EksNodeSecurityGroupIngress2:
    Type: AWS::EC2::SecurityGroupIngress
    DependsOn: EksNodeSecurityGroup
    Properties:
      IpProtocol: tcp
      FromPort: 1025
      ToPort: 65535 
      GroupId: !Ref EksNodeSecurityGroup
      SourceSecurityGroupId: !Ref EksServiceSecurityGroup 

  # open 443 to Service SG
  EksNodeSecurityGroupIngress3:
    Type: AWS::EC2::SecurityGroupIngress
    DependsOn: EksNodeSecurityGroup
    Properties:
      IpProtocol: tcp
      FromPort: 443
      ToPort: 443 
      GroupId: !Ref EksNodeSecurityGroup
      SourceSecurityGroupId: !Ref EksServiceSecurityGroup

### Node Security group Egress ###
  # All open
  EksNodeSecurityGroupEgress1:    
    Type: AWS::EC2::SecurityGroupEgress
    DependsOn: EksNodeSecurityGroup
    Properties: 
      FromPort: -1
      ToPort: -1 
      GroupId: !Ref EksNodeSecurityGroup
      IpProtocol: -1
      CidrIp: 0.0.0.0/0 
```
- The last resource we need is a SSH key the cluster will use to encrypt data at rest. We will use AWS SSM to create store and manage accessibility to this key. This key will be used by the EKs cluster to encrypt the secrets and configurations in it. The key has access defnied on the admin user and root. We can add more users to this key based on the use. Example, we have a user on AWS Lambda who might query the EKS cluster for some data, that user will need access to this key on certain access (List,Decrypt,etc) to be able to read the data from the EKS cluster.
```
## SSH Key for cluster secret encryption
  EksKMS:
    Type: AWS::KMS::Key
    Properties:
      Description: Key for EKS cluster to use when encrypting your Kubernetes secrets
      KeyPolicy:
        Version: '2012-10-17'
        Id: EKS-key
        Statement:
        - Sid: Enable IAM User Permissions
          Effect: Allow
          Principal:
            AWS: !Sub arn:aws:iam::${AWS::AccountId}:root
          Action: kms:*
          Resource: '*'
        - Sid: Allow administration of the key
          Effect: Allow
          Principal:
            AWS: !Sub 
            - 'arn:aws:iam::${AWS::AccountId}:user/${AWSaccountAdminUserName}'
            - {AWSaccountAdminUserName: !Ref AWSaccountAdminUserName}
          Action:
          - kms:Create*
          - kms:Describe*
          - kms:Enable*
          - kms:List*
          - kms:Put*
          - kms:Update*
          - kms:Revoke*
          - kms:Disable*
          - kms:Get*
          - kms:Delete*
          - kms:ScheduleKeyDeletion
          - kms:CancelKeyDeletion
          Resource: '*'

  EksKMSAlias:
    Type: AWS::KMS::Alias
    Properties: 
      AliasName: !Ref EksKMSAliasName
      TargetKeyId: !GetAtt EksKMS.Arn
```
This complete set of resource will spin up as a single Cloudformation stack . Please fork my github file here.

### EKS Compute Resource
This section deploys the actual bare-metals of your cluster(**hah Gotcha! there is no bare metal, its AWS, its upon some cloud in the sky**). 
We deploy the service section of EKS and after that the worker node cluster of EKS. Every parameter that we use here comes from the last two templates. 
It is worth mentioning that the smallest t-shirt size of EC2 that you can use is t2.small and the smallest disk that can be attached to it is of 4GB . 
```
## EKS Build ##
  EKS:
    Type: AWS::EKS::Cluster
    Properties:
      Name: !Ref EKSClusterName
      ResourcesVpcConfig:
        SecurityGroupIds: !Ref EksServiceSecurityGroupID
        SubnetIds: !Ref SubnetIds
      RoleArn: !Ref EksClusterServiceRole
      Version: !Ref KubernetesVersion

  EKSNodegroup:
    Type: AWS::EKS::Nodegroup
    DependsOn: EKS
    Properties:
      AmiType: !Ref AmiType
      ClusterName: !Ref EKS
      DiskSize: !Ref DiskSize
      ForceUpdateEnabled: !Ref ForceUpdateEnabled
      InstanceTypes:
        - !Ref InstanceTypes
      NodegroupName: !Ref EKSManagedNodeGroupName
      NodeRole: !Ref EksClusterNodeRoleIP
      ScalingConfig:
        MinSize: !Ref NodeAutoScalingGroupMinSize
        DesiredSize: !Ref NodeAutoScalingGroupDesiredSize
        MaxSize: !Ref NodeAutoScalingGroupMaxSize
      Subnets: !Ref SubnetIds
      RemoteAccess:
        Ec2SshKey: !Ref Ec2SshKey
```
This complete set of resource will spin up as a single Cloudformation stack . Please fork my github file here.

---
The cluster takes about 10-15 mins to be ready. To access the cluster, we will need to install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) on our local system. Kubectl is a command line tool for controlling Kubernetes clusters, allowing you to run commands against Kubernetes clusters. You can use kubectl to deploy applications, inspect and manage cluster resources, and view logs.

We need to configure the kubectl to talk to our EKS cluster. For this we need to go and edit the file **~./kube/config** with the details of the cluster we find on the EKS tab of the AWS console. The format of the file will be as such
```
clusters:
- cluster:
    certificate-authority-data: << TOKEN >>
    server: << ENDPOINT URL >>
  name: << ARN OF THE CLUSTER >>
contexts:
- context:
    cluster: << ARN OF THE CLUSTER >>
    user: << ARN OF THE CLUSTER >>
  name: << CLUSTER ID >>
current-context: << CLUSTER ID >>
kind: Config
preferences: {}
users:
- name: << CLUSTER ID >>
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      args:
      - token
      - -i
      - << CLUSTER ID >>
      - -r
      - << ADMIN USER ROLE ARN >>
      command: aws-iam-authenticator
      env:
      - name: AWS_PROFILE
        value: << AWS PROFILE >>
```
**NOTE:** This is error prone and a much easier way of doing it will be leveragin the aws cli and its admin role profile to modify this file for you by executing the following :
```
aws eks update-kubeconfig --name my-eks-cluster --profile <ADMIN-PROFILE> --region <EKS-Cluster-Region>
```
If everything is good till this point, you should be good to hit a sane kubectl command to see whats the fuss it all about.
```
kubectl get namespace
```
### WARNING:  
**Enjoy your K8s responsibly.
Please dont keep this setup running. Cloudformations are super easy to recreate, so take this down once you are done for the day. Mr.Bezos has enough $$$s as we speak.** 