# AWS User Creation.

When we open an account in AWS, we are the root user. While you can do do any things with that user, It is important to follow best practice and create groups and users for each type of access you might need.
In my case, I was spinning up an EKS cluster. I thought it gives me an opportunity to write a small blog on how to go about it.
We can create resource and query them through AWS API using AWS CLI. The CLI gives us a lot of power to do things through API. But the CLI needs to get access. These access can be classified into users and each account can thus have multiple users and access. Instead of directly giving access to the users, we put users into a group. These groups then get access using policies. Each group can have one or more users.

Each **USER** is created within a certain **GROUP** and each group is given access to resources dictated by **POLICIES**.This way, you can edit access to multiple users.

In my use case, I am creating an admin group for the entire aws account and an EKS admin group just for EKS service:

Both of these can be created using cloud formation through your root account:

The following create a admin account:

[Admin.yaml](/UsersGroups/Admin.yaml)

This user is suppose to have access through console, hence i provided the username and password.

The second user is for programmatic access to EKS API:

[EKSAdmin.yaml :](/UsersGroups/EKSAdmin.yaml)

- Create Cloudformation stacks with these files to create the resources.
- These files creates a group , user and a policy to attach to them.
- To access the user from a local terminal, we need to configure the keys to the user a an AWS profile
- After the role is created, Go to : AWS > IAM > Users > [User] > Security Credentials
- Create **Access Keys**
- Copy the **Access Key ID** and **Secret Access Key**
- Open in your terminal:
```
nano ~/.aws/configure

$credentials
[SOME-PROFILE-NAME]
aws_access_key_id=<COPIED FROM AWS>
aws_secret_access_key=<COPIED FROM AWS>
```
- After configuring both user accounts, my configure file looks as such

- My configure file looks as such:
```
[p-admin]
aws_access_key_id=FAKEFAKEFAKEFAKEFAKEFAKEFAKEFAKE
aws_secret_acess_key=FAKEFAKEFAKEFAKEFAKEFAKEFAKEFAKE

[p-eks]
aws_access_key_id=FAKEFAKEFAKEFAKEFAKEFAKEFAKEFAKE
aws_secret_access_key=FAKEFAKEFAKEFAKEFAKEFAKEFAKEFAKE
```

- After the profile is configured, we can make aws cli command referencing the profiles:

```
aws eks list-clusters --profile [p-eks] --region [REGION-NAME]
```

The AWS Admin profile can be used to login to the portal as such:

- Go to : https://aws.amazon.com/console/
- Choose to login as IAM user and NOT root user
- Once logged, please setup a **Multi-Factor Authentication (MFA)** at:
```
IAM > User > [User Name]> [Security Credentials] > Assigned MFA device > Virtual MFA device
```



### Published on:
https://dev.to/anupamncsu/aws-access-through-users-groups-3253