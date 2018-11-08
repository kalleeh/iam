# Installation Instructions

In order to use this application to deploy IAM policies throughout your organisation you must first create the param files for your respective accounts.

1. Update the `IAMAccount` param to reflect the AccountId of your AWS account that holds your IAM users and groups. If you are using Federation update the `SAML Provider` param with the name of your SAML IdP instead.
2. Update the `DeploymentAccount` param to reflect the AccountId of your AWS account that holds your deployment pipelines.
3. Create a params file for each of your accounts where regular users will need access to. In this params file you may consider setting the following paramaters:
    * `"isDevelopmentAccount": "true"` - creates a developer role
    * `"isPowerUserRequired": "true"` - creates a power user role
    * `"isNetworkedAccount": "true"` - creates a network admin role
    * `"isReadOnlyRequired": "true"` - creates a readonly role
    * `"isPartnerAdminAccessRequired": "false"` - prevents Partner from having admin access
4. Once complete, remember to update the deployment-map.yml file and associated pipeline params.json file under the deployment account to include the iam-baseline application.
5. Financial Admin roles need to be created manually on the master account.
    * Execute the fin-admin.yml cloudformation template while logged in under the master account.
        ```bash
        aws cloudformation create-stack --stack-name gov-iam --template-body file://fin-admin.yml --parameters ParameterKey=IAMAccount,ParameterValue=<AccountID> ParameterKey=StackPrefix,ParameterValue=gov ParameterKey=FinanceAdminRoleName,ParameterValue=financeadmin --region eu-central-1 --capabilities CAPABILITY_NAMED_IAM
        ```
    * Enable IAM access to billing information by following these [instructions](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/grantaccess.html). Note: You will need to be logged in as the root user on the master account.

Note: If the deployment-map.yml was pushed into the git repository _after_ this template.yml then then iam-baseline pipeline will need to be manually retriggered from the Deployment account

## Access Controls

* All IAM Users and Groups will be managed through the IAM account
* Users will need to authenticate against the IAM account and then assume roles to other accounts as needed
* Only IAM Administrators will have access to manage IAM Users and Groups.
* IAM Roles and Policies will be deployed to organization accounts through the pipeline framework

### IAM Groups

The groups below are created under the IAM account. With the exception of the iam-administrators group, all groups have an inline policy which allows members of the group to assume a role in another account. For this to work the target account must trust the IAM account.

|Group|Roles which can be assumed|Description|
|-----|--------------------------|-----------|
|[gov-deploymentadmin](#deployment-admins)|gov-deploymentadmin-role|Administer deployment pipelines
|[gov-deploymentviewer](#deployment-viewers)|gov-deploymentviewer-role|View deployment pipelines
|[gov-developer](#developer-and-power-user-roles)|gov-developer-role|Read/write access for developers
|[gov-financeadmin](#finance-administrator)|gov-financeadmin-role|Billing Information
|[gov-iam-admins](#iam-admninistrator)|n/a|Manage IAM resources
|[gov-infosecaudit](#infosec-read-only)|gov-infosecaudit-role|Audit and monitor accounts
|[gov-networkadmin](#network-administrators)|gov-networkadmin-role|Network resources including DX
|[gov-partneradmin](#partner-admins)|gov-partneradmin-role|Full Administrative access to all accounts
|[gov-poweruser](#developer-and-power-user-roles)|gov-poweruser-role|Read/write access for operations
|[gov-readonly](#read-only-role)|gov-readonly-role|Read-only access to all resources

### IAM Roles

|Role|Policies|
|----|--------|
|gov-deploymentadmin-role|gov-deploymentadmin-policy|
|gov-deploymentviewer-role|gov-deploymentviewer-policy|
|gov-developer-role|gov-poweruser-policy|
|gov-financeadmin-role|gov-financeadmin-policy|
|gov-infosec-role|ReadOnlyAccess|
|gov-networkadmin-role|gov-networkadmin-policy|
|gov-partneradmin-role|AdministratorAccess|
|gov-poweruser-role|gov-poweruser-policy|
|gov-readonly-role|ReadOnlyAccess|

#### Deployment Admins

Deployment admins will have access to the CI/CD services in the deployment account. Users assuming this role will be responsible for managing and approving deployment pipelines into other accounts.

Typically these users will interact through the deployment git repository updating the deployment map and pipeline configurations.

* Full access to CodePipeline
* Full access to CodeCommit
* Full access to CodeBuild
* Full access to CodeDeploy
* Full access to CloudFormation

### User Profile Descriptions

#### Deployment Viewers

Developers will usually only have access to their own development accounts, however in order to be able to see the status of their deployments through the pipeline they are also able to assume this role. With this role they have read-only access to all pipelines and codebuild projects. Logs will be visible for all built projects, however the code base for each project can not be seen through this role.

* Read-only access to Code Pipeline
* Read-only access to Code Build

#### Developer and Power User Roles

The Developer and Power User roles are essentially the same, however they have been split up because different users will assume different roles depending on their user profile. I.e. Developers will assume the developer role and will therefore receive the same set of permissions but only to developer accounts. Likewise Ops users will assume the Power Users role to access production accounts.

* Access to push to CodeCommit in the development account in order to kick off deployment pipelines. (Only applicable to development accounts)
* Access to create and manage resources in the developer account e.g. EC2, RDS, ECS etc.
* A developer will not have access to perform the following actions:
  * Create/Modify or Remove any resources which are a part of the deployment pipeline framework.
  * Create/Modify or Remove IAM roles or policies which have been created as part of the security and iam baselines (including this role)
  * Create IAM users in the development accounts

#### Finance Administrator

The Finance Admin role will be deployed to the master account and will be given access to view billing related information that is propagated up from the child accounts.

* Full access to Cost Explorer and Billing

#### IAM Administrator

An IAM Admin group will be created in the IAM account. Users in this group will have full IAM access and will generally be responsible in the short term to manage IAM users and groups. Each of the roles defined in the following slides will have a corresponding IAM group in the IAM account.

* Full IAM access in the IAM account

#### InfoSec Read-Only

This role is the same as the Read-Only Role however it is created on all accounts by default (including the Audit account) and only InfoSec users will have access to assume this role.

* Read-only access to the audit account which receives logs from other organisation accounts
* This role should be deployed to all accounts to give InfoSec full visibility of the environment.

#### Network Administrators

This role can be deployed to all accounts with networked resources and provides network administrators full access to resources such as VPCs, Subnets, Security Groups etc. While these resources should be created through the pipeline framework it may be required at times to make emergency changes in order to respond to security threats.

* Access to create/modify EC2 network resources. e.g. Security Groups
* Access to create/modify VPC resources. e.g. Subnets, ACLs, etc.
* Access to push to CodeCommit in a shared sercices account in order to kick off deployment pipelines.

#### Partner Admins

In order to allow Partner administrators to manage the AWS infrastructure a Partner Admin role is being deployed to all accounts by default. Partner admins will be able to assume this role.
Individual accounts can opt-out of this role where Partner administration is not required.

* Full Administrator access on all accounts (excluding exception accounts e.g. audit account)

#### Read-Only Role

This role can be used for most business unit OU accounts. For example it can be used in test or if desired production to allow developers to assist in troubleshooting issues.

* Read-only access to all resources e.g. EC2, RDS, ECS, CloudWatch Logs etc.

### Access Matrix

The table below is an example of how the groups and roles can be used to provide access to the different accounts within the organisation.

|                 |Master|Deployment|IAM|Audit|AppDev|AppTest|AppProd|
|-----------------|:----:|:--------:|:-:|:---:|:----:|:-----:|:-----:|
|Deployment Admin |-|RW|-|-|-|-|-
|Deployment Viewer|-|RO|-|-|-|-|-
|Developer        |-|-|-|-|RW|RO|RO
|Finance Admin    |RW<sup>1</sup>|-|-|-|-|-|-
|IAM Admin        |-|-|RW|-|-|-|-
|InfoSec          |-|RO|RO|RO|RO|RO|RO
|Network Admin    |-|-|-|-|RW|RW|RW
|Partner Admin  |-|RW|RW|-|RW|RW|RW
|Power User       |-|-|-|-|RW|RW|RW
|Read Only        |-|-|-|-|RO|RO|RO

<sup>1</sup> RW permissions are granted only to the finance related services
