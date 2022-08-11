  AWS Setup Guide — Puddle at H2O Installation and Administration 1.7.10 documentation         

[![](_static/logo.png)](index.html)

1.7.10

  

*   [Change Log](changelog.html)
*   [Installation](install.html)
    *   [Azure VPC Setup Guide](install-azure.html)
    *   [AWS Setup Guide](#)
        *   [AWS Resources](#aws-resources)
            *   [Prerequisites](#prerequisites)
            *   [Environment Variables Setup](#environment-variables-setup)
            *   [Network Setup](#network-setup)
            *   [PostgresSQL Setup](#postgressql-setup)
            *   [Redis Setup](#redis-setup)
            *   [IAM Setup](#iam-setup)
            *   [Puddle Backend EC2 Setup](#puddle-backend-ec2-setup)
            *   [AWS Resources Review](#aws-resources-review)
        *   [Puddle Application](#puddle-application)
            *   [Installation](#installation)
            *   [Reverse Proxy Setup](#reverse-proxy-setup)
            *   [Create the License File](#create-the-license-file)
            *   [Configuring Puddle](#configuring-puddle)
            *   [Configuring Environment Variables](#configuring-environment-variables)
            *   [Running Puddle](#running-puddle)
            *   [First Steps](#first-steps)
            *   [Stats Board (Optional)](#stats-board-optional)
    *   [GCP Setup Guide](install-gcp.html)
*   [Starting Puddle at H2O](starting.html)
*   [Administration](administration.html)

[Puddle at H2O Installation and Administration](index.html)

*   [Docs](index.html) »
*   [Installation](install.html) »
*   AWS Setup Guide
*   [View page source](_sources/install-aws.rst)

* * *

/\* CSS overrides for sphinx\_rtd\_theme \*/ /\* 24px margin \*/ .nbinput.nblast, .nboutput.nblast { margin-bottom: 19px; /\* padding has already 5px \*/ } /\* ... except between code cells! \*/ .nblast + .nbinput { margin-top: -19px; } .admonition > p:before { margin-right: 4px; /\* make room for the exclamation icon \*/ } /\* Fix math alignment, see https://github.com/rtfd/sphinx\_rtd\_theme/pull/686 \*/ .math { text-align: unset; }

AWS Setup Guide[¶](#aws-setup-guide "Permalink to this headline")
=================================================================

The following demonstrates how to deploy Puddle on AWS. The process involves steps for creating AWS resources via the `AWS CLI` as well as steps that will be taken on the Puddle Linux instance.

AWS Resources[¶](#aws-resources "Permalink to this headline")
-------------------------------------------------------------

The following section covers the steps to deploy and set up required AWS resources using shell script commands and AWS CLI.

### Prerequisites[¶](#prerequisites "Permalink to this headline")

*   Install the [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
*   You have permissions to do the following in your AWS account:
    
    *   Create a VPC with public and private subnets
    *   Create 2 Elastic IPs [\[1\]](#id3)
    *   Create an AWS ElastiCache Redis instance
    *   Create an AWS RDS Postgres instance
    *   Modify and create security groups
    *   Modify and create IAM roles
    
*   Make sure your choice of primary availability zone (`PRIMARY_AZ`) supports necessary GPU instance types to build DAI images [\[2\]](#id4)

[\[1\]](#id1)

Make sure you have at least two Elastic IP addresses available in the specified `REGION` in which you will be deploying.

[\[2\]](#id2)

You can use the [following steps](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-discovery.html) to search for the available instance types. Basically any NVIDIA CUDA GPU are sufficient. For instance to search for availability of instance type p3.something in AZ eu-central-1b:

aws ec2 describe-instance-type-offerings \\
    --location-type availability-zone \\
    --region eu-central-1 \\
    --output table \\
    --filters Name\=location,Values\=eu-central-1b \\
| grep "p3."

### Environment Variables Setup[¶](#environment-variables-setup "Permalink to this headline")

Set the following environment variables since these will be used throughout the whole deployment

REGION\=<deployment region> \# e.g. "eu-west-3"
PRIMARY\_AZ\=<primary availability zone> \# e.g. "eu-west-3a"
BACKUP\_AZ\=<backup availability zone> \# e.g. "eu-west-3b"
DEPLOYMENT\=<name of the deployment> \# e.g. "puddle-deployment"

Puddle application will run in a single region `REGION` in one availability zone `PRIMARY_AZ`. The second availability zone `BACKUP_AZ` is required only for Postgres instance (AWS RDS requires Postgres instance to be deployed into at least two different availability zones due to their backup strategy). The `DEPLOYMENT` variable will be used only as a prefix for easier identification of created resources.

### Network Setup[¶](#network-setup "Permalink to this headline")

In this section we will create the necessary network infrastructure for Puddle deployment.

Start with **creating VPC**:

\# Create VPC
VPC\="${DEPLOYMENT}\-vpc"
VPC\_ID\=$(aws ec2 create-vpc \\
    --tag-specifications "ResourceType=vpc,Tags=\[{Key=Name,Value=${VPC}}\]" \\
    --cidr-block "10.0.0.0/16" \\
    --region "${REGION}" \\
    --output text \\
    --query 'Vpc.VpcId')
echo "Created VPC ${VPC} (id=${VPC\_ID})"

Create **public subnet** (that will be exposed to the access from internet) and a **private subnet**:

\# Create public subnet
SUBNET\_PUBLIC\="${DEPLOYMENT}\-public-subnet"
SUBNET\_PUBLIC\_ID\=$(aws ec2 create-subnet \\
    --vpc-id "${VPC\_ID}" \\
    --cidr-block "10.0.1.0/24" \\
    --availability-zone "${PRIMARY\_AZ}" \\
    --tag-specifications "ResourceType=subnet,Tags=\[{Key=Name,Value=${SUBNET\_PUBLIC}}\]" \\
    --region "${REGION}" \\
    --output text \\
    --query 'Subnet.SubnetId')
echo "Created subnet ${SUBNET\_PUBLIC} (id=${SUBNET\_PUBLIC\_ID})"

\# Create private subnet
SUBNET\_PRIVATE\="${DEPLOYMENT}\-private-subnet"
SUBNET\_PRIVATE\_ID\=$(aws ec2 create-subnet \\
    --vpc-id "${VPC\_ID}" \\
    --cidr-block "10.0.2.0/24" \\
    --availability-zone "${PRIMARY\_AZ}" \\
    --tag-specifications "ResourceType=subnet,Tags=\[{Key=Name,Value=${SUBNET\_PRIVATE}}\]" \\
    --region "${REGION}" \\
    --output text \\
    --query 'Subnet.SubnetId')
echo "Created subnet ${SUBNET\_PRIVATE} (id=${SUBNET\_PRIVATE\_ID})"

Create **internet gateway** and attach it to the created VPC:

\# Create internet gateway
IGW\="${DEPLOYMENT}\-igw"
IGW\_ID\=$(aws ec2 create-internet-gateway \\
    --tag-specifications "ResourceType=internet-gateway,Tags=\[{Key=Name,Value=${IGW}}\]" \\
    --region "${REGION}" \\
    --output text \\
    --query 'InternetGateway.InternetGatewayId')
echo "Created internet gateway ${IGW} (id=${IGW\_ID}"

\# Attach internet gateway to VPC
RESULT\=$(aws ec2 attach-internet-gateway \\
    --internet-gateway-id "${IGW\_ID}" \\
    --vpc-id "${VPC\_ID}" \\
    --region "${REGION}")
echo "Attached internet gateway ${IGW} (id=${IGW\_ID}) to VPC ${VPC} (id=${VPC\_ID})"

Create **route table** for the public subnet to allow all outbound traffic from public subnet to access internet through the internet gateway created earlier:

\# Create route table for public subnet
RTB\_PUBLIC\="${DEPLOYMENT}\-rtb-public-subnet"
RTB\_PUBLIC\_ID\=$(aws ec2 create-route-table \\
    --vpc-id "${VPC\_ID}" \\
    --region "${REGION}" \\
    --tag-specifications "ResourceType=route-table,Tags=\[{Key=Name,Value=${RTB\_PUBLIC}}\]" \\
    --output text \\
    --query 'RouteTable.RouteTableId')
echo "Created route table ${RTB\_PUBLIC} (id=${RTB\_PUBLIC\_ID})"

\# Create route from public subnet to internet gateway
DESTINATION\="0.0.0.0/0"
RESULT\=$(aws ec2 create-route \\
    --region "${REGION}" \\
    --route-table-id "${RTB\_PUBLIC\_ID}" \\
    --destination-cidr-block "${DESTINATION}" \\
    --gateway-id "${IGW\_ID}")
echo "RouteTable ${RTB\_PUBLIC} (${RTB\_PUBLIC\_ID}): created route {destination=${DESTINATION}, target=${IGW\_ID}}"

\# Associate route table with public subnet
ASSOCIATION\_ID\=$(aws ec2 associate-route-table \\
    --region "${REGION}" \\
    --route-table-id "${RTB\_PUBLIC\_ID}" \\
    --subnet-id "${SUBNET\_PUBLIC\_ID}" \\
    --output text \\
    --query 'AssociationId')
echo "Associated route table ${RTB\_PUBLIC} (${RTB\_PUBLIC\_ID}) with subnet ${SUBNET\_PUBLIC} (${SUBNET\_PUBLIC\_ID}), associationId=${ASSOCIATION\_ID}"

Allocate 2 **Elastic IP addresses**

*   for NAT gateway
*   for Puddle backend instance

\# Allocate Elastic IP address for NAT gateway
EIP\_NAT\_GW\_ID\=$(aws ec2 allocate-address \\
    --domain vpc \\
    --region "${REGION}" \\
    --output text \\
    --query 'AllocationId')
\# Give the allocated Elastic IP address a name (Cannot be done during allocation)
EIP\_NAT\_GW\="${DEPLOYMENT}\-eip-nat-gw"
RESULT\=$(aws ec2 create-tags \\
    --resources "${EIP\_NAT\_GW\_ID}" \\
    --tags Key\=Name,Value\="${EIP\_NAT\_GW}" \\
    --region "${REGION}")
echo "Allocated Elastic IP address ${EIP\_NAT\_GW} (id=${EIP\_NAT\_GW\_ID})"

\# Allocate Elastic IP address for Puddle
EIP\_PUDDLE\_ID\=$(aws ec2 allocate-address \\
    --domain vpc \\
    --region "${REGION}" \\
    --output text \\
    --query 'AllocationId')
\# Give the allocated Elastic IP address a name (Cannot be done during allocation)
EIP\_PUDDLE\="${DEPLOYMENT}\-eip-puddle"
RESULT\=$(aws ec2 create-tags \\
    --resources "${EIP\_PUDDLE\_ID}" \\
    --tags Key\=Name,Value\="${EIP\_PUDDLE}" \\
    --region "${REGION}")
EIP\_PUDDLE\_IP\=$(aws ec2 describe-addresses \\
    --allocation-ids "${EIP\_PUDDLE\_ID}" \\
    --region "${REGION}" \\
    --output "text" \\
    --query 'Addresses\[0\].PublicIp')
echo "Allocated Elastic IP address ${EIP\_PUDDLE} (id=${EIP\_PUDDLE\_ID}, IP=${EIP\_PUDDLE\_IP})"

Create **NAT gateway** in public subnet and allow all outbound traffic from private subnet to access this NAT gateway:

\# Create NAT gateway
NAT\_GW\="${DEPLOYMENT}\-nat-gw"
NAT\_GW\_ID\=$(aws ec2 create-nat-gateway \\
    --allocation-id "${EIP\_NAT\_GW\_ID}" \\
    --region "${REGION}" \\
    --subnet-id "${SUBNET\_PUBLIC\_ID}" \\
    --tag-specifications "ResourceType=natgateway,Tags=\[{Key=Name,Value=${NAT\_GW}}\]" \\
    --output text \\
    --query 'NatGateway.NatGatewayId')
echo "Created NAT gateway (name=${NAT\_GW}, id=${NAT\_GW\_ID})"

\# Create route table for private subnet
RTB\_PRIVATE\="${DEPLOYMENT}\-rtb-private-subnet"
RTB\_PRIVATE\_ID\=$(aws ec2 create-route-table \\
    --vpc-id "${VPC\_ID}" \\
    --region "${REGION}" \\
    --tag-specifications "ResourceType=route-table,Tags=\[{Key=Name,Value=${RTB\_PRIVATE}}\]" \\
    --output text \\
    --query 'RouteTable.RouteTableId')
echo "Created route table ${RTB\_PRIVATE} (id=${RTB\_PRIVATE\_ID})"

\# Create route from private subnet to NAT gateway
DESTINATION\="0.0.0.0/0"
RESULT\=$(aws ec2 create-route \\
    --region "${REGION}" \\
    --route-table-id "${RTB\_PRIVATE\_ID}" \\
    --destination-cidr-block "${DESTINATION}" \\
    --nat-gateway-id "${NAT\_GW\_ID}")
echo "RouteTable ${RTB\_PRIVATE} (${RTB\_PRIVATE\_ID}): created route {destination=${DESTINATION}, target=${NAT\_GW\_ID}}"

\# Associate route table with private subnet
ASSOCIATION\_ID\=$(aws ec2 associate-route-table \\
    --region "${REGION}" \\
    --route-table-id "${RTB\_PRIVATE\_ID}" \\
    --subnet-id "${SUBNET\_PRIVATE\_ID}" \\
    --output text \\
    --query 'AssociationId')
echo "Associated route table ${RTB\_PRIVATE} (${RTB\_PRIVATE\_ID}) with subnet ${SUBNET\_PRIVATE} (${SUBNET\_PRIVATE\_ID}), associationId=${ASSOCIATION\_ID}"

Note

It may take some time until the NAT gateway is available. You can check in the background for its availability using the following snippet and proceed in the installation setup:

echo "Waiting for NAT gateway ${NAT\_GW} (${NAT\_GW\_ID}) to become available (this may take some time)..."
RESULT\=$(aws ec2 wait nat-gateway-available \\
    --nat-gateway-ids "${NAT\_GW\_ID}" \\
    --region "${REGION}")
echo "NAT gateway ${NAT\_GW} (${NAT\_GW\_ID}) is available"

Create **security group** `BACKEND_SG` that will be later used to allow access to Puddle backend instance via SSH and HTTPS:

\# Create security group for Puddle backend
BACKEND\_SG\="${DEPLOYMENT}\-puddle-backend-sg"
BACKEND\_SG\_ID\=$(aws ec2 create-security-group \\
    --group-name "${BACKEND\_SG}" \\
    --description "Puddle backend security group" \\
    --vpc-id "${VPC\_ID}" \\
    --region "${REGION}" \\
    --tag-specifications "ResourceType=security-group,Tags=\[{Key=Name,Value=${BACKEND\_SG}}\]" \\
    --output text \\
    --query 'GroupId')
echo "Created security group ${BACKEND\_SG} (id=${BACKEND\_SG\_ID})"

\# Add rules to the security group (for accessing Puddle instance later)
RESULT\=$(aws ec2 authorize-security-group-ingress \\
    --group-id "${BACKEND\_SG\_ID}" \\
    --ip-permissions \\
    IpProtocol\=tcp,FromPort\=22,ToPort\=22,IpRanges\='\[{CidrIp=0.0.0.0/0,Description="Allow SSH"}\]' \\
    IpProtocol\=tcp,FromPort\=443,ToPort\=443,IpRanges\='\[{CidrIp=0.0.0.0/0,Description="Allow HTTPS"}\]' \\
    --region "${REGION}")

Create **security group** `POSTGRES_SG` that will be later used for accessing Postgres from Puddle backend instance:

\# Create security group for Postgres instance
POSTGRES\_SG\="${DEPLOYMENT}\-postgres-sg"
POSTGRES\_SG\_ID\=$(aws ec2 create-security-group \\
    --group-name "${POSTGRES\_SG}" \\
    --description "Postgres security group" \\
    --vpc-id "${VPC\_ID}" \\
    --region "${REGION}" \\
    --tag-specifications "ResourceType=security-group,Tags=\[{Key=Name,Value=${POSTGRES\_SG}}\]" \\
    --output text \\
    --query 'GroupId')
echo "Created security group ${POSTGRES\_SG} (id=${POSTGRES\_SG\_ID})"

\# Allow access from backend\_sg to port 5432
RESULT\=$(aws ec2 authorize-security-group-ingress \\
    --group-id "${POSTGRES\_SG\_ID}" \\
    --ip-permissions \\
        "IpProtocol=tcp,FromPort=5432,ToPort=5432,UserIdGroupPairs=\[{GroupId=${BACKEND\_SG\_ID},Description=Allow port 5432 from backend\_sg}\]" \\
    --region "${REGION}")

Create **security group** `REDIS_SG` that will be later used for accessing Redis from Puddle backend instance:

\# Create security group for Redis instance
REDIS\_SG\="${DEPLOYMENT}\-redis-sg"
REDIS\_SG\_ID\=$(aws ec2 create-security-group \\
    --group-name "${REDIS\_SG}" \\
    --description "Redis security group" \\
    --vpc-id "${VPC\_ID}" \\
    --region "${REGION}" \\
    --tag-specifications "ResourceType=security-group,Tags=\[{Key=Name,Value=${REDIS\_SG}}\]" \\
    --output text \\
    --query 'GroupId')
echo "Created security group ${REDIS\_SG} (id=${REDIS\_SG\_ID})"

\# Allow access from backend\_sg to port 6379
RESULT\=$(aws ec2 authorize-security-group-ingress \\
    --group-id "${REDIS\_SG\_ID}" \\
    --ip-permissions \\
        "IpProtocol=tcp,FromPort=6379,ToPort=6379,UserIdGroupPairs=\[{GroupId=${BACKEND\_SG\_ID},Description=Allow port 6379 from backend\_sg}\]" \\
    --region "${REGION}")

Create **security group** `VM_SG` that will be later used for accessing all VMs from Puddle backend instance:

\# Create security group for managed VMs (DAI, H2O3)
VM\_SG\="${DEPLOYMENT}\-vm-sg"
VM\_SG\_ID\=$(aws ec2 create-security-group \\
    --group-name "${VM\_SG}" \\
    --description "VMs security group" \\
    --vpc-id "${VPC\_ID}" \\
    --region "${REGION}" \\
    --tag-specifications "ResourceType=security-group,Tags=\[{Key=Name,Value=${VM\_SG}}\]" \\
    --output text \\
    --query 'GroupId')
echo "Created security group ${VM\_SG} (id=${VM\_SG\_ID})"

\# Allow inbound traffic from puddle backend security group to VMs
RESULT\=$(aws ec2 authorize-security-group-ingress \\
    --group-id "${VM\_SG\_ID}" \\
    --ip-permissions \\
        "IpProtocol=tcp,FromPort=22,ToPort=22,UserIdGroupPairs=\[{GroupId=${BACKEND\_SG\_ID},Description=Allow SSH from backend\_sg}\]" \\
        "IpProtocol=tcp,FromPort=12345,ToPort=12345,UserIdGroupPairs=\[{GroupId=${BACKEND\_SG\_ID},Description=Allow port 12345 (DAI) from backend\_sg}\]" \\
        "IpProtocol=tcp,FromPort=54321,ToPort=54321,UserIdGroupPairs=\[{GroupId=${BACKEND\_SG\_ID},Description=Allow port 54321 (H2O3) from backend\_sg}\]" \\
    --region "${REGION}")

### PostgresSQL Setup[¶](#postgressql-setup "Permalink to this headline")

Next we create a PostgreSQL instance (an AWS RDS instance). AWS requires RDS instances to be created in a DB subnet group.

We will create a DB subnet group with two private subnets. We will reuse the already created private subnet (allocated in `PRIMARY_AZ`) and we will have to create another subnet in different availability zone (`BACKUP_AZ`).

Note

DB subnet group must have at least two subnets and these subnets must be in two different availability zones.

Create **private subnet** in different availability zone (`BACKUP_AZ`):

\# Create private subnet for backup data storage(s)
SUBNET\_DATA\_BACKUP\="${DEPLOYMENT}\-private-subnet-data-backup"
SUBNET\_DATA\_BACKUP\_ID\=$(aws ec2 create-subnet \\
    --vpc-id "${VPC\_ID}" \\
    --cidr-block "10.0.3.0/24" \\
    --availability-zone "${BACKUP\_AZ}" \\
    --tag-specifications "ResourceType=subnet,Tags=\[{Key=Name,Value=${SUBNET\_DATA\_BACKUP}}\]" \\
    --region "${REGION}" \\
    --output text \\
    --query 'Subnet.SubnetId')
echo "Created subnet ${SUBNET\_DATA\_BACKUP} (id=${SUBNET\_DATA\_BACKUP\_ID})"

Note

We will not create a custom route table for this subnet. When no custom route table is used, the main route table (route table that automatically comes with VPC) is associated with the subnet by default.

Create **DB subnet group** using two private subnet groups:

\# Create DB subnet group
DB\_SUBNET\_GROUP\="${DEPLOYMENT}\-db-subnet-group"
RESULT\=$(aws rds create-db-subnet-group \\
    --region "${REGION}" \\
    --db-subnet-group-name "${DB\_SUBNET\_GROUP}" \\
    --db-subnet-group-description "Subnet group for primary and backup postgres instances" \\
    --subnet-ids "${SUBNET\_PRIVATE\_ID}" "${SUBNET\_DATA\_BACKUP\_ID}")
echo "Created DB subnet group ${DB\_SUBNET\_GROUP}"

Before creating the DB instance, you have to specify **password** for master user. You can use the following snippet:

echo "Enter password (for db master user, at least 8 characters):"
read -s MASTER\_PASSWORD

Finally, when everything is set up you can create the **Postgres instance**:

\# Create Postgres db instance
DB\_INSTANCE\_IDENTIFIER\="${DEPLOYMENT}\-puddle"
RESULT\=$(aws rds create-db-instance \\
    --region "${REGION}" \\
    --db-name "puddle" \\
    --db-instance-identifier "${DB\_INSTANCE\_IDENTIFIER}" \\
    --db-instance-class "db.t2.medium" \\
    --engine "postgres" \\
    --engine-version "11.5" \\
    --master-username "puddle" \\
    --master-user-password "${MASTER\_PASSWORD}" \\
    --availability-zone "${PRIMARY\_AZ}" \\
    --db-subnet-group-name "${DB\_SUBNET\_GROUP}" \\
    --no-publicly-accessible \\
    --allocated-storage 50 \\
    --vpc-security-group-ids "${POSTGRES\_SG\_ID}")
echo "Created PostgreSQL database ${DB\_INSTANCE\_IDENTIFIER}"

Note

DB instance class determines computation and memory capacity of a DB instance. The recommended instance class for Puddle is `db.t2.medium` but you can choose another available from [AWS list of instance classes](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.DBInstanceClass.html)

Note

It may take some time until the Postgres instance is available. You can check in the background for its availability using the following snippet and proceed in the installation setup:

echo "Waiting for Postgres instance ${DB\_INSTANCE\_IDENTIFIER} to become available (this may take some time)..."
RESULT\=$(aws rds wait db-instance-available \\
    --db-instance-identifier "${DB\_INSTANCE\_IDENTIFIER}" \\
    --region "${REGION}")
echo "Postgres instance ${DB\_INSTANCE\_IDENTIFIER} is available"

### Redis Setup[¶](#redis-setup "Permalink to this headline")

In this section we will create a single-node, single-AZ Redis cache cluster. Redis cache cluster needs to be allocated in a cache subnet group.

Since we want a single-AZ cluster, we will create a new **cache subnet group** consisting only of one private subnet created at the beginning:

\# Create custom cache subnet group. This subnet group will contain only the
\# private subnet created earlier. Redis cluster will be placed into this subnet.
CACHE\_SUBNET\_GROUP\_NAME\="${DEPLOYMENT}\-cache-subnet-group"
RESULT\=$(aws elasticache create-cache-subnet-group \\
    --cache-subnet-group-name "${CACHE\_SUBNET\_GROUP\_NAME}" \\
    --cache-subnet-group-description "Cache subnet group for Puddle Redis" \\
    --subnet-ids "${SUBNET\_PRIVATE\_ID}" \\
    --region "${REGION}")
echo "Created cache subnet group ${CACHE\_SUBNET\_GROUP\_NAME}"

Second step is to create the **Redis cache cluster**:

\# Create single-node, single-AZ Redis cache cluster
CACHE\_CLUSTER\_ID\="${DEPLOYMENT}\-cache-cluster"
RESULT\=$(aws elasticache create-cache-cluster \\
    --cache-cluster-id "${CACHE\_CLUSTER\_ID}" \\
    --cache-subnet-group-name "${CACHE\_SUBNET\_GROUP\_NAME}" \\
    --engine "redis" \\
    --engine-version "4.0.10" \\
    --cache-node-type "cache.t2.medium" \\
    --num-cache-nodes 1 \\
    --region "${REGION}" \\
    --security-group-ids "${REDIS\_SG\_ID}")
echo "Created Redis cache cluster ${CACHE\_CLUSTER\_ID}"

Note

It may take some time until the cache cluster is available. You can check in the background for its availability using the following snippet and proceed in the installation setup:

echo "Waiting for Cache cluster ${CACHE\_CLUSTER\_ID} to become available (this may take some time)..."
RESULT\=$(aws elasticache wait cache-cluster-available \\
    --cache-cluster-id "${CACHE\_CLUSTER\_ID}" \\
    --region "${REGION}")
echo "Cache cluster ${CACHE\_CLUSTER\_ID} is available"

### IAM Setup[¶](#iam-setup "Permalink to this headline")

To manage access control of our EC2 instances, we need to create _instance profiles_ that will be holding roles with specified permissions. We will create two instance profiles:

*   for Puddle backend instance
*   for managed VMs

#### Puddle Backend IAM Setup[¶](#puddle-backend-iam-setup "Permalink to this headline")

In this section we will prepare an IAM role with a specified set of permissions that will be later attached as part of instance profile to the Puddle backend EC2 instance to control its access to another EC2 resources without sharing credentials.

First, get the AWS **account number** to scope the permissions by an account:

\# Use account number of the owner of VPC
ACCOUNT\=$(aws ec2 describe-vpcs \\
    --vpc-ids "${VPC\_ID}" \\
    --region "${REGION}" \\
    --output text \\
    --query 'Vpcs\[0\].OwnerId')

Next, prepare **policy document** for managing EC2 instances. Download [`puddle-backend-policy-template.json`](_downloads/puddle-backend-policy-template.json) file into your current directory and then run the following command to replace `${REGION}`, `${ACCOUNT}`, `${SUBNET_PRIVATE_ID}` and `${VM_SG_ID}` with correct values from your deployment:

sed \\
    -e "s/\\${REGION}/${REGION}/g" \\
    -e "s/\\${ACCOUNT}/${ACCOUNT}/g" \\
    -e "s/\\${SUBNET\_PRIVATE\_ID}/${SUBNET\_PRIVATE\_ID}/g" \\
    -e "s/\\${VM\_SG\_ID}/${VM\_SG\_ID}/g" \\
    puddle-backend-policy-template.json >puddle-backend-policy.json

As a result you will get a `puddle-backend-policy.json` file in your current directory with correct values that you will use in the following command to **create policy** for managing EC2 instances:

\# Create policy for managing EC2 instances (based on the prepared policy document)
POLICY\_NAME\="${DEPLOYMENT}\-puddle-backend-policy"
POLICY\_ARN\=$(aws iam create-policy \\
  --policy-name "${POLICY\_NAME}" \\
  --policy-document file://puddle-backend-policy.json \\
  --description "Puddle backend policy for managing EC2 instances" \\
  --output text \\
  --query 'Policy.Arn')
echo "Created policy ${POLICY\_NAME} (ARN=${POLICY\_ARN})"

To create the role we also have to prepare an **assume role policy document**. This assume role policy document will allow any EC2 instance to assume the puddle-backend-role.

Download [`role-trust-policy.json`](_downloads/role-trust-policy.json):

{
  "Version": "2012-10-17",
  "Statement": \[
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  \]
}

Create the role for Puddle backend instance:

Warning

`ROLE_NAME` must be unique within an AWS account

\# Create role for Puddle backend instance
ROLE\_NAME\="${DEPLOYMENT}\-puddle-backend-role"
ROLE\_ARN\=$(aws iam create-role \\
    --role-name "${ROLE\_NAME}" \\
    --assume-role-policy-document file://role-trust-policy.json \\
    --description "Role for Puddle backend instance to manage other EC2 instances" \\
    --output text \\
    --query 'Role.Arn')
echo "Created role ${ROLE\_NAME} (ARN=${ROLE\_ARN})"

Attach puddle-backend-policy to puddle-backend-role:

RESULT\=$(aws iam attach-role-policy \\
    --role-name "${ROLE\_NAME}" \\
    --policy-arn "${POLICY\_ARN}")
echo "Attached policy ${POLICY\_ARN} to role ${ROLE\_NAME}"

IAM role cannot be directly associated with an EC2 instance. Instead, we have to create an **IAM instance profile** that contains the created IAM role and provide this instance profile instead:

Warning

`INSTANCE_PROFILE_NAME` must be unique within an AWS account

\# Create instance profile for EC2 instance
INSTANCE\_PROFILE\_NAME\="${DEPLOYMENT}\-puddle-backend-instance-profile"
INSTANCE\_PROFILE\_ARN\=$(aws iam create-instance-profile \\
    --instance-profile-name "${INSTANCE\_PROFILE\_NAME}" \\
    --output text \\
    --query 'InstanceProfile.Arn')
echo "Created instance profile ${INSTANCE\_PROFILE\_NAME} (ARN=${INSTANCE\_PROFILE\_ARN})"

Finally, **add role to instance profile**:

RESULT\=$(aws iam add-role-to-instance-profile \\
    --role-name "${ROLE\_NAME}" \\
    --instance-profile-name "${INSTANCE\_PROFILE\_NAME}")

At this point you should have created an IAM instance profile with desired policies that will be later associated with the Puddle backend EC2 instance.

#### Managed VM IAM Setup[¶](#managed-vm-iam-setup "Permalink to this headline")

The process of IAM setup will be similar to the Puddle backend IAM setup, with the exception that we will not create specific policies for managed VMs. We will only:

1.  Create role
2.  Create instance profile
3.  Attach role to profile

**Create role** for managed VMs using the same `role-trust-policy.json` as for the Puddle backend role:

\# Create role for managed VM instances
MVM\_ROLE\_NAME\="${DEPLOYMENT}\-puddle-managed-vm-role"
MVM\_ROLE\_ARN\=$(aws iam create-role \\
    --role-name "${MVM\_ROLE\_NAME}" \\
    --assume-role-policy-document file://role-trust-policy.json \\
    --description "Role for Puddle managed VM instance" \\
    --output text \\
    --query 'Role.Arn')
echo "Created role ${MVM\_ROLE\_NAME} (ARN=${MVM\_ROLE\_ARN})"

**Create instance profile** for managed VM instances:

\# Create instance profile for managed VM instance
MVM\_INSTANCE\_PROFILE\_NAME\="${DEPLOYMENT}\-puddle-managed-vm-instance-profile"
MVM\_INSTANCE\_PROFILE\_ARN\=$(aws iam create-instance-profile \\
    --instance-profile-name "${MVM\_INSTANCE\_PROFILE\_NAME}" \\
    --output text \\
    --query 'InstanceProfile.Arn')
echo "Created instance profile ${MVM\_INSTANCE\_PROFILE\_NAME} (ARN=${MVM\_INSTANCE\_PROFILE\_ARN})"

**Add role to instance profile**:

RESULT\=$(aws iam add-role-to-instance-profile \\
    --role-name "${MVM\_ROLE\_NAME}" \\
    --instance-profile-name "${MVM\_INSTANCE\_PROFILE\_NAME}")

Now the instance profile `MVM_INSTANCE_PROFILE_NAME` is prepared and when a new managed VM is created by Puddle, it will be automatically associated with this instance profile. It’s up to you, what policies you attach to this role (using the same commands as when creating policies for Puddle backend role).

### Puddle Backend EC2 Setup[¶](#puddle-backend-ec2-setup "Permalink to this headline")

In this section we will create the Puddle backend EC2 instance.

First we have to create a **key-pair** to be later able to access the EC2 instance. The following command will

*   create and save a public key on AWS
*   create and return a private key (which we save into `PRIVATE_KEY_PATH`)

Warning

`KEY_NAME` must be unique within an AWS account

KEY\_NAME\=<key-pair-name> \# e.g. your AWS username
PRIVATE\_KEY\_PATH\=<path-to-pk-on-local> \# e.g. ~/.ssh/"${KEY\_NAME}.pem"

\# Create key pair (public key will be saved on AWS, private key is returned)
aws ec2 create-key-pair \\
    --key-name "${KEY\_NAME}" \\
    --output text \\
    --query 'KeyMaterial' \\
    --region "${REGION}" \\
    >"${PRIVATE\_KEY\_PATH}"

\# AWS requires private key to not be publicly viewable when using it later
chmod 400 "${PRIVATE\_KEY\_PATH}"
echo "Created key-pair ${KEY\_NAME}. Private key saved to ${PRIVATE\_KEY\_PATH}"

Note

If you want to import your own public key into AWS then you can follow this [AWS user guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#how-to-generate-your-own-key-and-import-it-to-aws).

Next we configure what AMI we want to run and with what resources, these are the recommended values:

\# Ubuntu Server 18.04 LTS (HVM), SSD Volume Type
IMAGE\_ID\=$(aws ec2 describe-images \\
    --owners 099720109477 \\
    --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-bionic-18.04-amd64-server-????????" "Name=state,Values=available" \\
    --query "reverse(sort\_by(Images, &CreationDate))\[:1\].ImageId" \\
    --output text)
INSTANCE\_TYPE\="c5d.xlarge" \# 4 CPUs (x86\_64), 8 GB memory, 100 GB storage (SSD)

At this point we have everything prepared to create the EC2 instance using the following command:

INSTANCE\_NAME\="${DEPLOYMENT}\-puddle-backend-instance"
INSTANCE\_ID\=$(aws ec2 run-instances \\
    --image-id "${IMAGE\_ID}" \\
    --instance-type "${INSTANCE\_TYPE}" \\
    --subnet-id "${SUBNET\_PUBLIC\_ID}" \\
    --security-group-ids "${BACKEND\_SG\_ID}" \\
    --key-name "${KEY\_NAME}" \\
    --region "${REGION}" \\
    --tag-specifications "ResourceType=instance,Tags=\[{Key=Name,Value=${INSTANCE\_NAME}}\]" \\
    --iam-instance-profile "Name=${INSTANCE\_PROFILE\_NAME}" \\
    --output text \\
    --query 'Instances\[0\].InstanceId')
echo "Created EC2 instance ${INSTANCE\_NAME} (id=${INSTANCE\_ID})"

Last step is to associate the earlier created Elastic IP address with the newly created EC2 instance:

\# Associate EIP with instance
ASSOCIATION\_ID\=$(aws ec2 associate-address \\
    --instance-id "${INSTANCE\_ID}" \\
    --allocation-id "${EIP\_PUDDLE\_ID}" \\
    --region "${REGION}" \\
    --output text \\
    --query 'AssociationId')
echo "Associated EIP ${EIP\_PUDDLE} (${EIP\_PUDDLE\_ID}) with instance ${INSTANCE\_NAME} (${INSTANCE\_ID})"

At this point the EC2 instance should be running and is accessible via SSH.

### AWS Resources Review[¶](#aws-resources-review "Permalink to this headline")

At the end you should have setup all AWS resources necessary for the following installation of Puddle application:

*   VPC
*   Three subnets
    
    *   Public subnet with internet gateway to enable direct communication over internet
    *   Private subnet for all other resources that should not be able to directly access internet (but they can do so using NAT gateway)
    *   Additional private subnet from different availability zone (necessary only for PostgreSQL instance)
    
*   Route tables with rules for inbound and outbound traffic in subnets
*   Internet gateway
*   NAT gateway
*   Two Elastic IP addresses
    
    *   one assigned to NAT gateway
    *   one assigned to Puddle backend EC2 instance
    
*   Three security groups
    
    *   SG for Puddle backend instance
    *   SG for Postgres and Redis
    *   SG for EC2 instances managed by Puddle backend instance
    
*   PostgreSQL database
*   Redis cache cluster
*   IAM role (for Puddle backend EC2 instance)
*   Key pair (for Puddle backend EC2 instance)
*   EC2 instance running Ubuntu server for Puddle backend application

Puddle Application[¶](#puddle-application "Permalink to this headline")
-----------------------------------------------------------------------

In this section we will setup Puddle application using the prepared AWS resources.

Connect to the created EC2 instance via SSH:

\# local-terminal>
ssh -i "${PRIVATE\_KEY\_PATH}" "ubuntu@${EIP\_PUDDLE\_IP}"

### Installation[¶](#installation "Permalink to this headline")

Install **Ansible**, **redis-cli**, **psql** and the **Puddle application**:

sudo apt update
sudo apt upgrade
sudo apt\-add-repository --yes --update ppa:ansible/ansible
sudo apt install -y wget unzip redis-tools postgresql-client ansible
wget https://s3.amazonaws.com/puddle-release.h2o.ai/1.7.10/x86\_64-ubuntu18/puddle\_1.7.10\_amd64.deb
sudo apt install -y ./puddle\_1.7.10\_amd64.deb

sudo bash
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum install epel-release ansible @postgresql:10 redis wget
wget https://s3.amazonaws.com/puddle-release.h2o.ai/1.7.10/x86\_64-centos7/puddle-1.7.10-1.x86\_64.rpm
rpm -i puddle-1.7.10-1.x86\_64.rpm

\[**Optional**\] Test that you can **connect to Redis** (this will work only if TLS is disabled for Redis):

redis-cli -h <redis\_address> PING
\# Confirm that the returned value is "PONG"

Test that you can **connect to PostgreSQL database**:

pg\_isready -h <postgres\_address> -p 5432
\# Confirm that the database is "accepting connections"

### Reverse Proxy Setup[¶](#reverse-proxy-setup "Permalink to this headline")

In this section we will set up **Traefik reverse proxy**.

#### Generate certificates[¶](#generate-certificates "Permalink to this headline")

First generate **self signed certificate** for use as root Certificate Authority:

sudo mkdir -p /opt/h2oai/puddle/certs/ca
cd /opt/h2oai/puddle/certs/ca
sudo openssl genrsa -out key.pem 4096
sudo openssl req -x509 -new -nodes -key key.pem -sha256 -days 1825 -out cert.pem -subj '/CN=Pudle - H2O.ai'

Warning

You could get the following warning when running the `openssl genrsa` command:

Can't load /home/ubuntu/.rnd into RNG
...RAND\_load\_file:Cannot open file...

This is caused by the missing default random file. **You can ignore this warning** (or comment out RANDFILE variable in /etc/ssl/openssl.cnf config file to get rid of the warning.)

Generate **server certificate**:

sudo mkdir -p /opt/h2oai/puddle/certs/server
cd /opt/h2oai/puddle/certs/server
sudo openssl genrsa -out key.pem 4096
\# Make sure to specify correct CN when asked, in this case it should be localhost
\# You may get a warning about a missing random file (you can ignore it)
sudo openssl req -new -key key.pem -out localhost.csr -subj '/CN=localhost'
sudo openssl x509 -req -in localhost.csr -CA ../ca/cert.pem -CAkey ../ca/key.pem -CAcreateserial -out cert.pem -days 1825

Generate **client certificate**:

sudo mkdir -p /opt/h2oai/puddle/certs/client
cd /opt/h2oai/puddle/certs/client
sudo openssl genrsa -out key.pem 4096
\# You may get a warning about a missing random file (you can ignore it)
sudo openssl req -new -key key.pem -out traefik.csr -subj '/CN=Puddle Traefik - H2O.ai'
sudo openssl x509 -req -in traefik.csr -CA ../ca/cert.pem -CAkey ../ca/key.pem -CAcreateserial -out cert.pem -days 1825

#### Configure Traefik[¶](#configure-traefik "Permalink to this headline")

When the certificates are prepared, we can **configure Traefik**:

*   Put external certificate (certificate shown to users, must be trusted) to `/opt/h2oai/puddle/certs/proxy-public/cert.pem`
*   Put the corresponding private key to `/opt/h2oai/puddle/certs/proxy-public/key.pem`
*   Both files have to be PEM encoded, password protection is not supported

#### Start Traefik[¶](#start-traefik "Permalink to this headline")

sudo systemctl stop puddle         \# make sure Puddle is stopped before starting traefik
sudo systemctl enable puddle-proxy \# start Traefik on boot
sudo systemctl start puddle-proxy  \# start Traefik
journalctl -u puddle-proxy | less  \# check that everything is ok

At this point Traefik reverse proxy should be running. In [Configure Traefik - Puddle part](#configure-traefik-puddle-part) we will finish configuration of Traefik so Puddle will be configured to use this running Traefik reverse proxy.

### Create the License File[¶](#create-the-license-file "Permalink to this headline")

1.  ssh into the Virtual Machine.
2.  Create a file `/opt/h2oai/puddle/license.sig` containing the license. Different path might be used, but this is the default.

### Configuring Puddle[¶](#configuring-puddle "Permalink to this headline")

We will need to fill the `/etc/puddle/config.yaml` file:

redis:
  connection:
    protocol: tcp
    address:
    password:
    tls: true

db:
  connection:
    drivername: postgres
    host:
    port: 5432
    user:
    dbname: puddle
    sslmode: require
    password:

tls:
  certFile: /opt/h2oai/puddle/certs/server/cert.pem
  keyFile: /opt/h2oai/puddle/certs/server/key.pem

connection:
  port: 8081

license:
  file: /opt/h2oai/puddle/license.sig

ssh:
  publicKey: /opt/h2oai/puddle/ssh/id\_rsa.pub
  privateKey: /opt/h2oai/puddle/ssh/id\_rsa

auth:
  token:
    secret:
  apiKeys:
    enabled: true
  activeDirectory:
    enabled: false
    server:
    port: 389
    baseDN:
    security: tls
    objectGUIDAttr: objectGUID
    displayNameAttr: displayName
    administratorsGroup: Puddle-Administrators
    usersGroup: Puddle-Users
    implicitGrant: false
  azureAD:
    enabled: false
    useAADLoginExtension: true
  awsCognito:
    enabled: false
    userPoolId:
    userPoolWebClientId:
    domain:
    redirectSignIn:
    redirectSignOut:
    adminsGroup: Puddle-Administrators
    usersGroup: Puddle-Users
    implicitGrant: false
  ldap:
    enabled: false
    host:
    port: 389
    skipTLS: false
    useSSL: true
    insecureSkipVerify: false
    serverName:
    baseDN:
    bindDN:
    bindPassword:
    bindAllowAnonymousLogin: false
    authenticationFilter: "(uid=%s)"
    authorizationFlow: userAttribute
    authorizationFilter: "(memberOf=%s)"
    authorizationSearchValueAttribute: dn
    uidAttributeName: uid
    uidNumberAttributeName: uidnumber
    emailAttributeName: email
    implicitGrant: false
    adminsGroup: Puddle-Administrators
    usersGroup: Puddle-Users
  oidc:
    enabled: false
    issuer:
    clientId:
    clientSecret:
    redirectUrl: /oidc/authorization-code/callback
    logoutUrl:
    scopes:
      \- openid
      \- profile
      \- email
      \- offline\_access
    implicitGrant: false
    adminRole: Puddle-Administrators
    userRole: Puddle-Users
    tokenRefreshInterval: 15m
    clientBearerTokenAuth:
      enabled: false
      issuer:
      clientId:
      scopes:
        \- openid
        \- offline\_access

packer:
  path: /opt/h2oai/puddle/deps/packer
  usePublicIP: true
  buildTimeoutHours: 1
  imageNameFormat: '%s'
  nvidiaDriversURL:
  terraformURL:
  driverlessAIURLPrefix:
  h2o3URLPrefix:

terraform:
  path: /opt/h2oai/puddle/deps/terraform
  usePublicIP: true
  pluginDir: /opt/h2oai/puddle/deps/terraform\_plugins/

reverseProxy:
  enabled: true
  port: 443
  caCertificate: /opt/h2oai/puddle/certs/ca/cert.pem
  caPrivateKey: /opt/h2oai/puddle/certs/ca/key.pem
  clientCertificate: /opt/h2oai/puddle/certs/client/cert.pem
  clientPrivateKey: /opt/h2oai/puddle/certs/client/key.pem

backend:
  baseURL:
  connections:
    usePublicIP: true

webclient:
  usePublicIP: true
  userSshAccessEnabled: true

providers:
  azure:
    enabled: false
    authority:
    location:
    rg:
    vnetrg:
    vnet:
    sg:
    subnet:
    enterpriseApplicationObjectId:
    adminRoleId:
    publicIpEnabled: true
    packerInstanceType:
    sshUsername: puddle
    sourceSharedImageGallery:
      subscriptionId:
      rg:
      name:
      imageName:
      imageVersion:
    sourceImageRG:
    sourceImageName:
    sourceImagePublisher: Canonical
    sourceImageOffer: UbuntuServer
    sourceImageSku: 16.04-LTS
    imageTags:
    customDataScriptPath:
    preflightScriptPath:
    packerCustomDataScriptPath:
    packerPreflightScriptPath:
    packerPostflightScriptPath:
    storageDiskLun: 0
    storageDiskFileSystem: ext4
    storageDiskDevice: /dev/sdc
    vmNamePrefix: puddle-
    vmNameRegexp: "^\[-0-9a-z\]\*\[0-9a-z\]$" \# starts with -, number or letter, ends with number or letter
    vmOwnerTagKey:
    vmTags:
      \# foo: bar
  aws:
    enabled: false
    owner:
    vpcId:
    sgIds:
    subnetId:
    iamInstanceProfile:
    publicIpEnabled: true
    packerInstanceType:
    encryptEBSVolume: true
    ebsKMSKeyArn:
    metadataEndpointIAMRole: http://169.254.169.254/latest/meta-data/iam/info
    suppressIAMRoleCheck: false
    sshUsername: ubuntu
    sourceAMIOwner: '099720109477'
    sourceAMINameFilter: ubuntu/images/\*ubuntu-xenial-16.04-amd64-server-\*
    packerRunTags:
    amiTags:
    userDataScriptPath:
    preflightScriptPath:
    packerUserDataScriptPath:
    packerPreflightScriptPath:
    packerPostflightScriptPath:
    storageEBSDeviceName: /dev/sdf
    storageDiskFileSystem: ext4
    storageDiskDevice: /dev/nvme1n1
    storageDiskDeviceGpu: /dev/xvdf
    vmNamePrefix: puddle-
    vmNameRegexp: "^\[-0-9a-z\]\*\[0-9a-z\]$" \# starts with -, number or letter, ends with number or letter
    vmOwnerTagKey:
    vmTags:
      \# foo: bar
  gcp:
    enabled: false
    project:
    zone:
    network: default
    subnetwork: ""
    publicIpEnabled: true
    encryptVolume: true
    volumeKmsKeyId: ""
    volumeKmsKeyRingName:
    volumeKmsKeyRingLocation:
    serviceAccountEmail: ""
    sshUsername: puddle
    storageDiskFileSystem: ext4
    startupScriptPath: ""
    preflightScriptPath: ""
    imageLabels:
    packerServiceAccountEmail: ""
    packerSourceImageProject: ubuntu-os-cloud
    packerSourceImageFamily: ubuntu-1604-lts
    packerInstanceType: n1-highmem-8
    packerAcceleratorType: nvidia-tesla-v100
    packerAcceleratorCount: 1
    packerStartupScriptPath: ""
    packerPreflightScriptPath: ""
    packerPostflightScriptPath: ""
    packerRunLabels:
    packerRunNetworkTags:
    vmNamePrefix: puddle-
    vmNameRegexp: "^\[-0-9a-z\]\*\[0-9a-z\]$" \# starts with -, number or letter, ends with number or letter
    runNetworkTags:
    backendServiceAccountEmail:
    useOsLogin: false
    vmOwnerTagKey:
    vmTags:
      \# foo: bar

products:
  dai:
    configTomlTemplatePath: "/opt/h2oai/puddle/configs/dai/config.toml"
    license:
    authType: local
    openid:
      baseURL:
      configurationURL:
      introspectionURL:
      authURLSuffix: "/auth"
      tokenURLSuffix: "/token"
      userinfoURLSuffix: "/userinfo"
      endSessionURLSuffix: "/logout"
      clientId:
      clientSecret:
      scope:
        \- openid
        \- profile
        \- email
      usernameFieldName: name
      userinfoAuthKey: sub
      clientTokens:
        clientId:
        issuer:
        scope:
          \- openid
          \- offline\_access
    googleAnalytics:
      usageStatsOptIn: true
      exceptionTrackerOptIn: false
      autodlMessagesTrackerOptIn: true
  h2o3:
    authEnabled: true

logs:
  dir: /opt/h2oai/puddle/logs
  maxSize: 1000
  maxBackups: 15
  maxAge: 60
  compress: true
  level: trace
  colored: true

mailing:
  enabled: true
  server:
  username:
  password:
  fromAddress: puddle@h2o.ai
  fromName: Puddle
  recipients:
  offsetHours: 24

idleTimeout:
  options:
    30min: 30
    1h: 60
    2h: 120
    3h: 180
    4h: 240
    Never: \-1

Description of all configuration fields can be found in the subsection [Configuration fields overview](#configuration-fields-overview).

#### Configuration fields overview[¶](#configuration-fields-overview "Permalink to this headline")

Below is the description of all configuration fields:

> *   Values for `redis.connection.*` can be found in following way:
>     
>     *   **Microsoft Azure**:
>     
>     > *   Search for **Azure Cache for Redis**.
>     > *   Select the newly created Redis instance.
>     > *   Select **Access keys**.
>     
>     *   **Amazon AWS**:
>     
>     > *   Go to **ElastiCache Dashboard**.
>     > *   Select **Redis**.
>     > *   Select the cluster used by Puddle.
>     > *   Select **Description** tab.
>     
>     *   **Google GCP**:
>     
>     > *   Go to **Memorystore > Redis**
>     > *   Select the instance used by Puddle.
>     > *   See **Connection properties**
>     
>     *   For `redis.connection.address` use format <hostname:port> (e.g. redis.xyk5qk.0001.euw3.cache.amazonaws.com:6379)
> *   `redis.workersCount` number of workers to spin up. It must be a positive integer and is 10 by default
>     
> *   Values for `db.connection.*` can be found in following way:
>     
>     *   **Microsoft Azure**:
>     
>     > *   Search for **Azure Database for PostgreSQL servers**.
>     > *   Select the newly created PostgreSQL instance.
>     > *   Select **Connection strings**.
>     > *   Use the password that was provided when creating the PostgreSQL database.
>     
>     *   **Amazon AWS**:
>     
>     > *   Go to **Amazon RDS**.
>     > *   Select **Databases**.
>     > *   Select the database used by Puddle.
>     
>     *   **Google GCP**:
>     
>     > *   Go to **SQL**
>     > *   Select the instance used by Puddle.
>     > *   See **Connect to this instance**
>     
>     *   For `db.connection.host` use only <hostname of Postgres instance> (e.g. postgres.cibba0yphezo.eu-west-3.rds.amazonaws.com)
> *   `tls.certFile` should point to the PEM encoded **certificate** file if you want to use HTTPS. If you don’t want to use HTTPS, leave this property empty. If you set this property, then `tls.keyFile` must be set as well.
>     
> *   `tls.keyFile` should point to the PEM encoded **private key** file if you want to use HTTPS. The private key **must be not encrypted by password**. If you don’t want to use HTTPS, leave this property empty. If you set this property, then `tls.certFile` must be set as well.
>     
> *   `connection.port` should be the port where Puddle backend should be running. Defaults to 80 or 443 based on TLS config.
>     
> *   `license.file` should be a path to the file containing the license (created in previous step).
>     
> *   `ssh.publicKey` should be the path to ssh public key (for example /opt/h2oai/puddle/ssh/id\_rsa.pub), which will be used by Puddle to talk to the Systems. **If this ssh key is changed, Puddle won’t be able to talk to the Systems created with old key, and these will have to be destroyed.**
>     
> *   `ssh.privateKey` should be the path to ssh private key (for example /opt/h2oai/puddle/ssh/id\_rsa), which will be used by Puddle to talk to the Systems. **If this ssh key is changed, Puddle won’t be able to talk to the Systems created with old key, and these will have to be destroyed.**
>     
> *   `auth.token.secret` should be a random string. It is used to encrypt the tokens between the backend and frontend. For example the following could be used to generate the secret:
>     
>     > tr -cd '\[:alnum:\]' < /dev/urandom | fold -w32 | head -n1
>     
> *   `auth.apiKeys.enabled` should be true/false and is true by default. If true the clients can authenticate using API Keys.
>     
> *   `auth.activeDirectory.enabled` should be true/false and is false by default. If true then authentication using ActiveDirectory is enabled.
>     
> *   `auth.activeDirectory.server` should be the hostname of the ActiveDirectory server, for example puddle-ad.h2o.ai.
>     
> *   `auth.activeDirectory.port` should be the port where ActiveDirectory is accessible, defaults to 389.
>     
> *   `auth.activeDirectory.baseDN` should be the BaseDN used for search.
>     
> *   `auth.activeDirectory.security` should be the security level used in communication with AD server. Could be none, start\_tls, tls, defaults to tls.
>     
> *   `auth.activeDirectory.objectGUIDAttr` should be the name of the attribute used as ID of the user, defaults to objectGUID.
>     
> *   `auth.activeDirectory.displayNameAttr` should be the name of the attribute used to determine groups where user is member, defaults to memberOf.
>     
> *   `auth.activeDirectory.administratorsGroup` should be the name of the Administrators group. Users in this group are assigned Administrator role in Puddle, users in Administrators group and Users group are considered Administrators.
>     
> *   `auth.activeDirectory.usersGroup` should be the name of the Users group. Users in this group are assigned User role in Puddle, users in Administrators group and Users group are considered Administrators.
>     
> *   `auth.activeDirectory.implicitGrant` should be true/false and is false by default. If true, then users are allowed access to Puddle (using user role) even if they are not members of Administrators nor Users group. If false, then users must be members of at least one group to be allowed access to Puddle.
>     
> *   `auth.azureAD.enabled` should be true/false and is false by default. If true, then authentication using Azure Active Directory is enabled. Also, if true, you have to set `provider.azure.authority`, `provider.azure.enterpriseApplicationObjectId`, and `provider.azure.adminRoleId` (`provider.azure.enabled` can be remain false); you also have to set `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID`, and `AZURE_CLIENT_ID` (see below).
>     
> *   `auth.azureAD.useAADLoginExtension` should be true/false and is false by default. If true, then ssh access to provisioned Virtual machines will use the Azure AD for authentication. Check [https://docs.microsoft.com/en-us/azure/virtual-machines/linux/login-using-aad](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/login-using-aad) for more information. Cannot be enabled, if using proxy for egress.
>     
> *   `auth.awsCognito.enabled` should be true/false and is false by default. If true, then authentication using AWS Cognito is enabled.
>     
> *   `auth.awsCognito.userPoolId` should be the Pool Id, for example us-east-1\_SlxxxxML1.
>     
> *   `auth.awsCognito.userPoolWebClientId` should be the App client id. The App client id can be found in following way:
>     
>     > *   Go to the AWS Cognito User Pool used by Puddle.
>     > *   Select the **App client settings**.
>     > *   Use the value under **ID**.
>     
> *   `auth.awsCognito.domain` should be the domain of the AWS Cognito User Pool, for example puddle.auth.<REGION>.amazoncognito.com (no https:// in the beginning). The domain can be found in following way:
>     
>     > *   Go to the AWS Cognito User Pool used by Puddle.
>     > *   Select the **Domain name**.
>     
> *   `auth.awsCognito.redirectSignIn` should be [https:/](https:/)/<SERVER\_ADDRESS>/aws-cognito-callback, please replace <SERVER\_ADDRESS> with hostname where Puddle is running.
>     
> *   `auth.awsCognito.redirectSignOut` should be [https:/](https:/)/<SERVER\_ADDRESS>/logout, please replace <SERVER\_ADDRESS> with hostname where Puddle is running.
>     
> *   `auth.awsCognito.adminsGroup` should be the name of a group in AWS Cognito User Pool. If users are members of this group, they are assigned Administrator role in Puddle.
>     
> *   `auth.awsCognito.usersGroup` should be the name of a group in AWS Cognito User Pool. If users are members of this group, they are assigned User role in Puddle.
>     
> *   `auth.awsCognito.implicitGrant` should be true/false and is false by default. If true, then users are allowed access to Puddle (using user role) even if they are not members of Administrators nor Users group. If false, then users must be members of at least one group to be allowed access to Puddle.
>     
> *   `auth.ldap.enabled` should be true/false and is false by default. If true, then authentication using LDAP is enabled.
>     
> *   `auth.ldap.host` should be the LDAP server hostname.
>     
> *   `auth.ldap.port` should be the port where LDAP is accessible, defaults to 389.
>     
> *   `auth.ldap.skipTLS` should be true/false and is false by default. If true then do not use TLS.
>     
> *   `auth.ldap.useSSL` should be true/false and is true by default. If true, then use SSL.
>     
> *   `auth.ldap.insecureSkipVerify` should be true/false and is false by default. If true, then skip the server’s certificate verification.
>     
> *   `auth.ldap.serverName` should be the server name from server’s certificate. Defaults to auth.ldap.host.
>     
> *   `auth.ldap.baseDN` should be the BaseDN where authentication search will start.
>     
> *   `auth.ldap.bindDN` should be the BindDN used by Puddle to query LDAP.
>     
> *   `auth.ldap.bindPassword` should be the password of the user used by Puddle to query LDAP.
>     
> *   `auth.ldap.bindAllowAnonymousLogin` should be true/false and is false by default. If true, then bind won’t be executed before getting user’s groups.
>     
> *   `auth.ldap.authenticationFilter` should be the filter used when authenticating user. Defaults to `"(uid=%s)"`.
>     
> *   `auth.ldap.authorizationFlow` should be userAttribute | groupAttribute. Defaults to userAttribute. Based on the value, either attribute of group (for example member) of attribute of user (for example memberOf) will be used in authorization.
>     
> *   `auth.ldap.authorizationFilter` should be the filter used when querying user’s groups. Defaults to `"(memberOf=%s)"`.
>     
> *   `auth.ldap.authorizationSearchValueAttribute` should be name of the attribute used during authorization. Defaults to `dn`.
>     
> *   `auth.ldap.uidAttributeName` should be the name of the uid attribute. Defaults to uid. Value of uid attribute cannot be empty. Values of uid and uid number create unique identifier of the user.
>     
> *   `auth.ldap.uidNumberAttributeName` should be the name of the uid number attribute. Defaults to uidnumber. Value of uid number attribute cannot be empty. Values of uid and uid number create unique identifier of the user.
>     
> *   `auth.ldap.emailAttributeName` should be the name of the email attribute. Defaults to email. Value of email attribute might be empty.
>     
> *   `auth.ldap.implicitGrant` should be true/false and is false by default. If true, then users are allowed access to Puddle (using user role) even if they are not members of Administrators nor Users group. If false, then users must be members of at least one group to be allowed access to Puddle.
>     
> *   `auth.ldap.adminsGroup` should be the name of the Administrators group. Users in this group are assigned Administrator role in Puddle, users in Administrators group and Users group are considered Administrators.
>     
> *   `auth.ldap.usersGroup` should be the name of the Users group. Users in this group are assigned User role in Puddle, users in Administrators group and Users group are considered Administrators.
>     
> *   `auth.ldap.cloudResourcesTagsMapping` should be a mapping from LDAP attributes to tags of provisioned cloud resources. The values of the specified LDAP tags are used as values for the specified cloud tags. User cannot change these and they are applied to every system a user launches.
>     
> *   `auth.oidc.enabled` should be true/false and is false by default. If true, then authentication using OpenID Connect is enabled.
>     
> *   `auth.oidc.issuer` should be the issuer of the tokens.
>     
> *   `auth.oidc.clientId` should be the clientId used by Puddle.
>     
> *   `auth.oidc.clientSecret` optional clientSecret used by Puddle.
>     
> *   `auth.oidc.redirectUrl` should be the redirect URL, defaults to /oidc/authorization-code/callback.
>     
> *   `auth.oidc.logoutUrl` should be the URL used to sign out users. end\_session\_endpoint value from the /.well-known/openid-configuration should be used.
>     
> *   `auth.oidc.scopes` should be the list of required scopes, defaults to openid, profile, email, offline\_access
>     
> *   `auth.oidc.implicitGrant` should be true/false and is false by default. If true, then users are allowed access to Puddle (using user role) even if they are not members of Administrators nor Users group. If false, then users must be members of at least one group to be allowed access to Puddle.
>     
> *   `auth.oidc.adminRole` should be the name of the Administrators role. Users with this role are assigned Administrator role in Puddle, users with Administrator role and User role are considered Administrators.
>     
> *   `auth.oidc.userRole` should be the name of the Users role. Users with this role are assigned User role in Puddle, users with Administrator role and User role are considered Administrators.
>     
> *   `auth.oidc.tokenRefreshInterval` should be the interval how often the OAuth2 tokens should be refreshed. Defaults to 15m.
>     
> *   `auth.oidc.clientBearerTokenAuth.enabled` should be true/false and is false by default. If true, then clients authentication using bearer token is enabled. If enabled, all of the auth.oidc.clientBearerTokenAuth.\* is required.
>     
> *   `auth.oidc.clientBearerTokenAuth.issuer` should be the issuer of the tokens.
>     
> *   `auth.oidc.clientBearerTokenAuth.clientId` should be the clientId used by Puddle when validating the client tokens. This client must support PKCE flow.
>     
> *   `auth.oidc.clientBearerTokenAuth.scopes` should be the list of required scopes, defaults to openid, offline\_access.
>     
> *   `packer.path` should point to the packer binary. Defaults to `/opt/h2oai/puddle/deps/packer`.
>     
> *   `packer.usePublicIP` should be true/false and is true by default. If true then packer will create VMs with public IP addresses, otherwise private IP will be used.
>     
> *   `packer.buildTimeoutHours` should be the number of hours after which the packer build times out. Default is 1 hour.
>     
> *   `packer.imageNameFormat` optional format string used to compute the name of the images. The format string has to contain exactly one %s placeholder. The %s will be substituted by <PRODUCT>-<VERSION>-<TIMESTAMP>. For example if the format string is cust-%s-image and Puddle is building Driverless AI image of version 1.8.0, then the name of the resulting image will be cust-dai-1.8.0-<TIMESTAMP>-image.
>     
> *   `packer.nvidiaDriversURL` if some custom URL for downloading NVIDIA drivers is required, for example because of internet access restrictions, set it here. Make sure to use version 440.59. Defaults to [http://us.download.nvidia.com/XFree86/Linux-x86\_64/440.59/NVIDIA-Linux-x86\_64-440.59.run](http://us.download.nvidia.com/XFree86/Linux-x86_64/440.59/NVIDIA-Linux-x86_64-440.59.run). For local files use [file:///absolute-path/to/required.file](file:///absolute-path/to/required.file).
>     
> *   `packer.terraformURL` if custom URL for downloading Terraform is required, for example because of internet access restrictions, set it here. Make sure to use version 0.11.14. Defaults to [https://releases.hashicorp.com/terraform/0.11.14/terraform\_0.11.14\_linux\_amd64.zip](https://releases.hashicorp.com/terraform/0.11.14/terraform_0.11.14_linux_amd64.zip). For local files use [file:///absolute-path/to/required.file](file:///absolute-path/to/required.file).
>     
> *   `packer.driverlessAIURLPrefix` if custom URL for downloading Driverless AI is required, for example because of internet access restrictions, set it here. For local directory containing Driverless AI installers use [file:///absolute-path/to/dir/](file:///absolute-path/to/dir/).
>     
> *   `packer.h2o3URLPrefix` if custom URL for downloading H2O-3 is required, for example because of internet access restrictions, set it here. For local directory containing H2O-3 installers use [file:///absolute-path/to/dir/](file:///absolute-path/to/dir/).
>     
> *   `terraform.path` should point to the terraform binary. Defaults to `/opt/h2oai/puddle/deps/terraform`.
>     
> *   `terraform.usePublicIP` should be true/false and is true by default. If true then terraform will use public IP to communicate with the provisioned Virtual machines, otherwise private IP will be used.
>     
> *   `terraform.pluginDir` optional path where Terraform plugins are stored. If set, Terraform will not try to download plugins during initialization.
>     
> *   `reverseProxy.enabled` should be true/false and is false by default. If true then reverse proxy is used.
>     
> *   `reverseProxy.port` should be port where reverse proxy is running, defaults to 1234.
>     
> *   `reverseProxy.caCertificate` should be path to the CA certificate used to issue HTTPS certificates for systems provisioned by Puddle. Must be a PEM encoded certificate with no password protection.
>     
> *   `reverseProxy.caPrivateKey` should be path to the CA private key. Must be a PEM encoded private key with no password protection.
>     
> *   `reverseProxy.clientCertificate` should be path to the certificate used when doing forward auth (relevant for H2O-3 systems). Must be a PEM encoded certificate with no password protection.
>     
> *   `reverseProxy.clientPrivateKey` should be path to the private key used when doing forward auth (relevant for H2O-3 systems). Must be a PEM encoded private key with no password protection.
>     
> *   `backend.baseURL` should be the URL where Puddle is running (including port), for example [https://puddle.h2o.ai:443](https://puddle.h2o.ai:443)
>     
> *   `backend.openFilesWarningThreshold` should be a number and defaults to 300. If more than this number of files are open by Puddle, then an Alert is created in the UI.
>     
> *   `backend.connections.usePublicIp` should be true/false and is true by default. If true then backend will use public IP to communicate with the provisioned Virtual machines, otherwise private IP will be used.
>     
> *   `webclient.usePublicIp` should be true/false and is true by default. If true then public IP is shown in UI, otherwise private IP is displayed.
>     
> *   `webclient.outOfCUsUserMsg` should be a message displayed to users when they try to create or start a system, but there are not enough Compute Units available.
>     
> *   `webclient.userSshAccessEnabled` should be true/false and is true by default. If true then users are able to download SSH keys of the provisioned VMs.
>     
> *   `providers.azure.enabled` should be true/false and is false by default. If true then Microsoft Azure is enabled as provider in Puddle. All variables under `providers.azure` must be set if enabled.
>     
> *   `providers.azure.authority` should be set to `https://login.microsoftonline.com/<Azure ActiveDirectory Name>.onmicrosoft.com`. The Azure Active Directory name can be found in following way:
>     
>     > *   Go to **Azure Active Directory** blade.
>     > *   Select **Overview**.
>     
> *   `providers.azure.location` should be set to the same value that was specified for the Resource group, for example `eastus`.
>     
> *   `providers.azure.rg` should be set to the **name** of the newly created Resource group.
>     
> *   `providers.azure.vnetrg` should be set to the **name** of the Resource group where VNET and Subnet are present.
>     
> *   `providers.azure.vnet` should be set to the **id** of the newly created Virtual network.
>     
> *   `providers.azure.sg` should be set to the **id** of the newly created Network security group.
>     
> *   `providers.azure.subnet` should be set to the **id** of the newly created Subnet.
>     
> *   `providers.azure.enterpriseApplicationObjectId` should be the Object ID of the Enterprise Application. The Enterprose Application Object ID can be found in following way:
>     
>     > *   Go to the **Azure Active Directory** blade.
>     > *   Select **Enterprise Applications**.
>     > *   Select the newly created Enterprise Application.
>     > *   Use the **Object ID**.
>     
> *   `providers.azure.adminRoleId` should be set to the ID of the newly created Administator Role in the Application Registration Manifest. The Administator Role ID can be found in following way:
>     
>     > *   Go to the **Azure Active Directory** blade.
>     > *   Select **App registrations (preview)**.
>     > *   Select the newly created App registration.
>     > *   Select **Manifest**.
>     > *   Search for **Administator** role under **appRoles** and use the ID of this role.
>     
> *   `providers.azure.publicIpEnabled` should be true/false and is true by default. Public IP is created if and only if this is set to true. Must be set to true if at least one of packer, terraform, backend or webclient uses public IP.
>     
> *   `providers.azure.packerInstanceType` should be the instance type used by Packer to build images. Defaults to Standard\_NC6.
>     
> *   `providers.azure.sshUsername` should be the username used when doing SSH/SCP from backend. Defaults to puddle.
>     
> *   `providers.azure.sourceSharedImageGallery.subscriptionId` ID of the subscription where the Shared Image Gallery with image used as source is present. Leave empty or remove if other image source should be used.
>     
> *   `providers.azure.sourceSharedImageGallery.rg` name of the resource group where the Shared Image Gallery with image used as source is present. Leave empty or remove if other image source should be used.
>     
> *   `providers.azure.sourceSharedImageGallery.name` name of the Shared Image Gallery with image used as source is present. Leave empty or remove if other source should be used.
>     
> *   `providers.azure.sourceSharedImageGallery.imageName` name of the Image from Shared Image Gallery to use as source. Leave empty or remove if other source should be used.
>     
> *   `providers.azure.sourceSharedImageGallery.imageVersion` version of the Image from Shared Image Gallery to use as source. Leave empty or remove if other source should be used.
>     
> *   `providers.azure.sourceImageRG` name of the resource group containing the private image used as the source for newly built Puddle images. Leave empty or remove if other source should be used.
>     
> *   `providers.azure.sourceImageName` name of the private image which should be used as the source for newly built Puddle images. Leave empty or remove if other source should be used.
>     
> *   `providers.azure.sourceImagePublisher` ignored if other source image is set as well (for example private image, or image from Shared Image Gallery). Should be the name of the publisher of the image used as source for newly built Puddle images. Defaults to Canonical. Leave empty or remove if other source should be used.
>     
> *   `providers.azure.sourceImageOffer` ignored if other source image is set as well (for example private image, or image from Shared Image Gallery). Should be the name of the offering of the publisher used as source for newly build Puddle images. Defaults to UbuntuServer. Leave empty or remove if other source should be used.
>     
> *   `providers.azure.sourceImageSku` ignored if other source image is set as well (for example private image, or image from Shared Image Gallery). Should be the image sku used as source for newly built Puddle images. Defaults to 16.04-LTS. Leave empty or remove if other source should be used.
>     
> *   `providers.azure.imageTags` map of tags used for all Packer resources and produced Image.
>     
> *   `providers.azure.customDataScriptPath` optional path to script with custom data to supply to the machine when provisioning new system. This can be used as a cloud-init script.
>     
> *   `providers.azure.preflightScriptPath` optional path to script which will be executed during System provisioning by Puddle. This is not a cloud-init script, but a shell script which is executed after cloud init is finished.
>     
> *   `providers.azure.packerVmNames` optional list of VM names used by Packer. Should be used, if custom naming policies are enforced. Defaults to list from “packer-builder-01” to “packer-builder-10”. Number of elements in this list determines how many images can be built in parallel.
>     
> *   `providers.azure.packerCustomDataScriptPath` optional path to script with custom data to supply to the machine when building new image. This can be used as a cloud-init script.
>     
> *   `providers.azure.packerPreflightScriptPath` optional path to script which will be executed at the beginning of image build process. This is not a cloud-init script, but a shell script which is executed after cloud init is finished.
>     
> *   `providers.azure.packerRebootAfterPreflight` should be true/false and is false by default. If true, then the Packer VM is rebooted after executing preflight script (even if there no script configured).
>     
> *   `providers.azure.packerPostflightScriptPath` optional path to script which will be executed at the end of image build process. This is not a cloud-init script, but a shell script which is executed after cloud init is finished.
>     
> *   `providers.azure.packerRebootAfterPostflight` should be true/false and is false by default. If true, then the Packer VM is rebooted after executing postflight script (even if there no script configured).
>     
> *   `providers.azure.storageDiskLun` should be the LUN of the storage data disk, defaults to 0.
>     
> *   `providers.azure.storageDiskFileSystem` should be the filesystem used with storage data disk, default to ext4.
>     
> *   `providers.azure.storageDiskDevice` should be the path to the device used as a storage data disk, defaults to /dev/sdc.
>     
> *   `providers.azure.vmNamePrefix` should be prefix added to every VM name, might be empty, defaults to “puddle-“.
>     
> *   `providers.azure.vmNameRegexp` should be the regexp used to validate VM name (before the prefix is added). Defaults to “^\[-0-9a-z\]\*\[0-9a-z\]$”.
>     
> *   `providers.azure.vmNameRegexpDescription` should be the human-readable explanation of the regexp used to validate VM name. Defaults to “lowercase letters, numbers and hyphens only, cannot end with hyphen”.
>     
> *   `providers.azure.vmOwnerTagKey` should be a string and is empty by default, Unless empty, a tag will be added to the new VMs created by Puddle, the value is owner’s email. If owner has no email, the this tag is not added.
>     
> *   `providers.azure.vmTags` contains any additional tags which should be applied to the provisioned VMs.
>     
> *   `providers.aws.enabled` should be true/false and is false by default. If true then Amazon AWS is enabled as provider in Puddle. All variables under `providers.aws` must be set if enabled.
>     
> *   `providers.aws.owner` should be the owner of the newly created resources (if you followed AWS setup guide then use value of `${ACCOUNT}`).
>     
> *   `providers.aws.vpcId` should be the ID of the VPC where Virtual machines will be launched (if you followed AWS setup guide then use value of `${VPC_ID}`).
>     
> *   `providers.aws.sgIds` should be the list of IDs of the Security Groups applied to provisioned Virtual machines (if you followed AWS setup guide then use value of `${VM_SG_ID}`).
>     
> *   `providers.aws.subnetId` should be the ID of the Subnet where Virtual machines will be placed (if you followed AWS setup guide then use value of `${SUBNET_PRIVATE_ID}`).
>     
> *   `providers.aws.iamInstanceProfile` should be the name of the IAM Instance Profile assigned to the Virtual machines (if you followed AWS setup guide then use value of `${MVM_INSTANCE_PROFILE_NAME}`).
>     
> *   `providers.aws.publicIpEnabled` should be true/false and is false by default. If true, then no public IP will be assigned. Must be set to true if at least one of packer, terraform, backend or webclient uses public IP.
>     
> *   `providers.aws.packerInstanceType` should be the instance type used by packer to build images, defaults to p3.2xlarge.
>     
> *   `providers.aws.encryptEBSVolume` should be true/false and is false by default. If true then EBS Volumes are encrypted using KMS Key. The KMS Key is unique for every system.
>     
> *   `` providers.aws.ebsKMSKeyArn` `` should be the arn of KMS key used to encrypt all VMs. If this is empty then a new KMS is created for every VM.
>     
> *   `providers.aws.metadataEndpointIAMRole` should be URL which is used to check assigned IAM role. Defaults to [http://169.254.169.254/latest/meta-data/iam/info](http://169.254.169.254/latest/meta-data/iam/info).
>     
> *   `providers.aws.suppressIAMRoleCheck` should be true/false and is false by default. If true then Puddle does not try to obtain assigned IAM role from AWS Metadata endpoint.
>     
> *   `providers.aws.sourceAMIOwner` owner of the AMI used as source for newly built Puddle AMIs. Defaults to 099720109477 (Canonical).
>     
> *   `providers.aws.sourceAMINameFilter` name of the image, with wildcards, which should be used as source for newly built Puddle images. Defaults to ubuntu/images/_ubuntu-xenial-16.04-amd64-server-_.
>     
> *   `providers.aws.packerRunTags` map of tags used for Packer EC2 instance and Volume. These tags are not applied to produced AMI.
>     
> *   `providers.aws.amiTags` map of tags used for AMIs built by Puddle.
>     
> *   `providers.aws.userDataScriptPath` optional path to script with custom user data script which should be used when launching new systems.
>     
> *   `providers.aws.preflightScriptPath` optional path to script which will be executed during System provisioning by Puddle. This is not a cloud-init script, but a shell script which is executed after cloud init is finished.
>     
> *   `providers.aws.packerVmNames` optional list of VM names used by Packer. Should be used, if custom naming policies are enforced. Defaults to list from “packer-builder-01” to “packer-builder-10”. Number of elements in this list determines how many images can be built in parallel.
>     
> *   `providers.aws.packerUserDataScriptPath` optional path to script with custom user data script which should be used when building new images.
>     
> *   `providers.aws.packerPreflightScriptPath` optional path to script which will be executed at the beginning of image build process. This is not a cloud-init script, but a shell script which is executed after cloud init is finished.
>     
> *   `providers.aws.packerRebootAfterPreflight` should be true/false and is false by default. If true, then the Packer VM is rebooted after executing preflight script (even if there no script configured).
>     
> *   `providers.aws.packerPostflightScriptPath` optional path to script which will be executed at the end of image build process. This is not a cloud-init script, but a shell script which is executed after cloud init is finished.
>     
> *   `providers.aws.packerRebootAfterPostflight` should be true/false and is false by default. If true, then the Packer VM is rebooted after executing postflight script (even if there no script configured).
>     
> *   `providers.aws.storageEBSDeviceName` should be the device name used when attaching EBS Volume, defaults to /dev/sdf
>     
> *   `providers.aws.storageDiskFileSystem` should be the filesystem used with storage data disk, default to ext4.
>     
> *   `providers.aws.storageDiskDevice` should be the path to the device used as a storage data disk, defaults to /dev/nvme1n1.
>     
> *   `providers.aws.vmNamePrefix` should be prefix added to every VM name, might be empty, defaults to “puddle-“.
>     
> *   `providers.aws.vmNameRegexp` should be the regexp used to validate VM name (before the prefix is added). Defaults to “^\[-0-9a-z\]\*\[0-9a-z\]$”.
>     
> *   `providers.aws.vmNameRegexpDescription` should be the human-readable explanation of the regexp used to validate VM name. Defaults to “lowercase letters, numbers and hyphens only, cannot end with hyphen”.
>     
> *   `providers.aws.disableFSR.checkInterval` interval how often should Puddle check available snapshots and disable their enabled FSRs. Defaults to 15 minutes.
>     
> *   `providers.aws.disableFSR.snapshotsBatchSize` how many snapshots should be checked in one batch. Defaults to 20.
>     
> *   `providers.aws.disableFSR.snapshotsBatchInterval` interval how long should Puddle wait before processing next snapshots batch. Defaults to 5 seconds.
>     
> *   `providers.aws.vmOwnerTagKey` should be a string and is empty by default, Unless empty, a tag will be added to the new VMs created by Puddle, the value is owner’s email. If owner has no email, the this tag is not added.
>     
> *   `providers.aws.vmTags` contains any additional tags which should be applied to the provisioned VMs.
>     
> *   `providers.gcp.enabled` should be true/false and is false by default. If true then Google GCP is enabled as provider in Puddle. At least the variables `providers.gcp.project` and `providers.gcp.zone` must be set if enabled.
>     
> *   `providers.gcp.project` should be id of the project that will host the newly created resources.
>     
> *   `providers.gcp.zone` should be the GCE Zone that will host the newly created resources.
>     
> *   `providers.gcp.network` should be the name of the network that will host the newly created VMs, defaults to “default”. Optional if `providers.gcp.subnetwork` is set (needs to point to the subnet’s network).
>     
> *   `providers.gcp.subnetwork` should the name of the subnetwork that will host the newly created VMs, required for custom subnetmode networks, has to be in a region that includes `providers.gcp.zone`, has to be in `providers.gcp.network` if specified (incl. the default), defaults to empty.
>     
> *   `providers.gcp.publicIpEnabled` should be true/false and is false by default. If true, then no public IP will be assigned to Puddle-managed VMs. Must be set to true if at least one of packer, terraform, backend or webclient uses public IP.
>     
> *   `providers.gcp.encryptVolume` should be true/false and is false by default. If true, all systems’ VM volumes will be encrypted. If true, either `providers.gcp.volumeKmsKeyId` or `providers.gcp.volumeKmsKeyRingName` and `providers.gcp.volumeKmsKeyRingLocation` have to be set.
>     
> *   `providers.gcp.volumeKmsKeyId` should be the full resource name of the key used to encrypt all VMs, for example projects/XXX/locations/XXX/keyRings/XXX/cryptoKeys/XXX. If empty and `providers.gcp.encryptVolume` is true then a new KMS is created for every system, defaults to empty.
>     
> *   `providers.gcp.volumeKmsKeyRingName` should be the name of the KMS key ring in which unique volume KMS keys will be created. Ignored if `providers.gcp.volumeKmsKeyId` is set.
>     
> *   `providers.gcp.volumeKmsKeyRingLocation` should be the location of the KMS key ring in which unique volume KMS keys will be created. Ignored if `providers.gcp.volumeKmsKeyId` is set.
>     
> *   `providers.gcp.serviceAccountEmail` should be the service account used for Puddle system VMs, uses the default GCE service account if empty, defaults to empty.
>     
> *   `providers.gcp.sshUsername` should be the username for SSH/SCP, defaults to “puddle”.
>     
> *   `providers.gcp.storageDiskFileSystem` should be the file system name used for storage disks (must be compatible with `mkfs` on the image), defaults to “ext4”.
>     
> *   `providers.gcp.startupScriptPath` optional path to script used as input for startup script (cloud init).
>     
> *   `providers.gcp.preflightScriptPath` optional path to script which will be executed during System provisioning by Puddle. This is not a cloud-init script, but a shell script which is executed after cloud init is finished.
>     
> *   `providers.gcp.imageLabels` should be map of labels to be applied to the images built by Puddle, see [https://cloud.google.com/compute/docs/labeling-resources](https://cloud.google.com/compute/docs/labeling-resources) for value restrictions.
>     
> *   `providers.gcp.packerVmNames` optional list of VM names used by Packer. Should be used, if custom naming policies are enforced. Defaults to list from “packer-builder-01” to “packer-builder-10”. Number of elements in this list determines how many images can be built in parallel.
>     
> *   `providers.gcp.packerServiceAccountEmail` should be the service account used for Puddle Packer VMs, uses the default GCE service account if empty, defaults to empty.
>     
> *   `providers.gcp.packerSourceImageProject` should be the project that hosts the image family to be used as source for newly built Puddle images, defaults to “ubuntu-os-cloud”.
>     
> *   `providers.gcp.packerSourceImageFamily` should be the image family that should be used as source for newly built Puddle images, defaults to “ubuntu-1604-lts”.
>     
> *   `providers.gcp.packerInstanceType` should be the machine type used by Packer to build images, defaults to “n1-highmem-8”.
>     
> *   `providers.gcp.packerAcceleratorType` should be the type of accelerators to be attached to the VM used by Packer to build images, defaults to “nvidia-tesla-v100”. Needs to be available in `providers.gcp.zone`.
>     
> *   `providers.gcp.packerAcceleratorCount` should be the number of accelerators to be attached to the VM used by Packer to build images, must be >= 0, defaults to 1.
>     
> *   `providers.gcp.packerStartupScriptPath` optional path to script used as input for startup script (cloud init) in packer VMs.
>     
> *   `providers.gcp.packerPreflightScriptPath` optional path to script which will be executed at the beginning of image build process. This is not a cloud-init script, but a shell script which is executed after cloud init is finished.
>     
> *   `providers.gcp.packerRebootAfterPreflight` should be true/false and is false by default. If true, then the Packer VM is rebooted after executing preflight script (even if there no script configured).
>     
> *   `providers.gcp.packerPostflightScriptPath` optional path to script which will be executed at the end of image build process. This is not a cloud-init script, but a shell script which is executed after cloud init is finished.
>     
> *   `providers.gcp.packerRebootAfterPostflight` should be true/false and is false by default. If true, then the Packer VM is rebooted after executing postflight script (even if there no script configured).
>     
> *   `providers.gcp.packerRunLabels` should be map of labels used for Packer VMs and Volumes, these labels are not applied to the resulting image, see [https://cloud.google.com/compute/docs/labeling-resources](https://cloud.google.com/compute/docs/labeling-resources) for value restrictions.
>     
> *   `providers.gcp.packerRunNetworkTags` should be list of network tags applied to Packer VMs. See [https://cloud.google.com/vpc/docs/add-remove-network-tags](https://cloud.google.com/vpc/docs/add-remove-network-tags) for more details.
>     
> *   `providers.gcp.vmNamePrefix` should be prefix added to every VM name, might be empty, defaults to “puddle-“.
>     
> *   `providers.gcp.vmNameRegexp` should be the regexp used to validate VM name (before the prefix is added). Defaults to “^\[-0-9a-z\]\*\[0-9a-z\]$”.
>     
> *   `providers.gcp.vmNameRegexpDescription` should be the human-readable explanation of the regexp used to validate VM name. Defaults to “lowercase letters, numbers and hyphens only, cannot end with hyphen”.
>     
> *   `providers.gcp.runNetworkTags` should be list of network tags applied to all VMs managed by Puddle (except Packer VMs, use `providers.gcp.packerRunTags` to configure network tags for Packer VMs).
>     
> *   `providers.gcp.backendServiceAccountEmail` should be the service account used for Puddle Backend VM. Must be set if providers.gcp.useOsLogin is true.
>     
> *   `providers.gcp.useOsLogin` should be true/false and is false by default. If true, OS Login is used for SSH connections. OS Login must be configured on project level before enabling this option.
>     
> *   `providers.gcp.vmOwnerTagKey` should be a string and is empty by default, Unless empty, a tag will be added to the new VMs created by Puddle, the value is owner’s email. If owner has no email, the this tag is not added.
>     
> *   `providers.gcp.vmTags` contains any additional tags which should be applied to the provisioned VMs.
>     
> *   `products.dai.configTomlTemplatePath` should be the path to custom config.toml file, which will be used as default configuration for all new Driverless AI Systems. If not set, the default file is used.
>     
> *   `products.dai.license` should be the path to DriverlessAI license file. If set, then this license will be automatically installed on all provisioned systems.
>     
> *   `products.dai.authType` should be local/openid and defaults to local. Local auth uses the htpasswd file injected by Puddle. OpenID auth uses the OpenID Connect. If openid is set, then all of the products.dai.openid.\* values are required.
>     
> *   `products.dai.openid.baseURL` should be the base url of all the OpenID endpoints. For example if the authorization endpoint is [https://example.com/auth/realms/master/protocol/openid-connect/auth](https://example.com/auth/realms/master/protocol/openid-connect/auth) then the baseURL should be [https://example.com/auth/realms/master/protocol/openid-connect](https://example.com/auth/realms/master/protocol/openid-connect).
>     
> *   `products.dai.openid.configurationURL` should be the absolute url of configuration endpoint, for example [https://example.com/auth/realms/master/.well-known/openid-configuration](https://example.com/auth/realms/master/.well-known/openid-configuration)
>     
> *   `products.dai.openid.introspectionURL` should be the absolute url of introspection endpoint, for example [https://example.com/auth/realms/master/protocol/openid-connect/token/introspect](https://example.com/auth/realms/master/protocol/openid-connect/token/introspect)
>     
> *   `products.dai.openid.authURLSuffix` should be the URL suffix for the authorization endpoint. It can be obtained from the configuration URL response.
>     
> *   `products.dai.openid.tokenURLSuffix` should be the URL suffix for the token endpoint. It can be obtained from the configuration URL response.
>     
> *   `products.dai.openid.userinfoURLSuffix` should be the URL suffix for the userinfo endpoint. It can be obtained from the configuration URL response.
>     
> *   `products.dai.openid.endSessionURLSuffix` should be the URL suffix for the logout endpoint. It can be obtained from the configuration URL response.
>     
> *   `products.dai.openid.clientId` should be the client id used by Driverless AI to query the identity provider.
>     
> *   `products.dai.openid.clientSecret` should be the client secret used by Driverless AI to query the identity provider.
>     
> *   `products.dai.openid.scope` should be array of required scopes. Usually \[openid, profile, email\] is sufficient.
>     
> *   `products.dai.openid.usernameFieldName` should be the name of the field in the ID token. Value of this field is then used as username.
>     
> *   `products.dai.openid.userinfoAuthKey` should be the name of the field in Access Token. Value of this field is then used to authorize the user access. Please note, that the value of this field must match the sub from ID token. In most cases this should be sub.
>     
> *   `products.dai.openid.clientTokens.clientId` should be the client id used by Driverless AI Python Client. This client must support PKCE flow without client secret.
>     
> *   `products.dai.openid.clientTokens.issuer` should be the issuer of tokens used by Driverless AI Python Client.
>     
> *   `products.dai.openid.clientTokens.scope` should be the scopes used by Driverless AI Python Client. Usually \[openid, offline\_access\] is sufficient.
>     
> *   `products.dai.googleAnalytics.usageStatsOptIn` should be true/false and is true by default. If true, opt-in for usage statistics and bug reporting.
>     
> *   `products.dai.googleAnalytics.exceptionTrackerOptIn` should be true/false and is false by default. If true, opt-in for full tracebacks tracking.
>     
> *   `products.dai.googleAnalytics.autodlMessagesTrackerOptIn` should be true/false and is true by default. If true, opt-in for experiment preview and summary messages tracking.
>     
> *   `products.h2o3.authEnabled` should be true/false and is false by default. If true the H2O-3 has Basic Auth enabled. Use of reverse proxy is recommended in this case to enable one-click login to H2O-3.
>     
> *   `logs.dir` should be set to a directory where logs should be placed.
>     
> *   `logs.maxSize` should be the max size of log file, in MB, defaults to 1000.
>     
> *   `logs.maxBackups` should be the number of old files retained, defaults to 15.
>     
> *   `logs.maxAge` should be the max age of retained files, in days, defaults to 60. Older files are always deleted.
>     
> *   `logs.compress` should be true/false and is true by default. If true then the files will be compressed when rotating.
>     
> *   `logs.level` log level to use, should be trace/debug/info/warning/error/fatal and is trace by default.
>     
> *   `logs.colored` should be true/false and is true by default. If true, logs will be color coded.
>     
> *   `mailing.enabled` should be true/false. If true then mailing is enabled. All fields under `mailing` are mandatory if this is set to true.
>     
> *   `mailing.server` should be the hostname and port of the SMTP server, for example smtp.example.com:587.
>     
> *   `mailing.username` should be the client username.
>     
> *   `mailing.password` should be the client password.
>     
> *   `mailing.fromAddress` should be the email address used as FROM, for example in case of an address ‘<Puddle> [puddle@h2o.ai](mailto:puddle%40h2o.ai)’ this field should be set to [puddle@h2o.ai](mailto:puddle%40h2o.ai).
>     
> *   `mailing.fromName` should be the name used as FROM, defaults to Puddle, for example in case of an address ‘<Puddle> [puddle@h2o.ai](mailto:puddle%40h2o.ai)’ this field should be set to Puddle.
>     
> *   `mailing.recipients` should be the space-separated list of recipients.
>     
> *   `mailing.offsetHours` should be a number of hours between repeated email notifications, defaults to 24, does not apply to FAILED system notifications.
>     
> *   `idleTimeout.options` should be a mapping from labels to values (in minutes) of possible idle timeout options. Use -1 as value for option to never time out.
>     

#### Configure Traefik - Puddle part[¶](#configure-traefik-puddle-part "Permalink to this headline")

In [Reverse Proxy Setup](#reverse-proxy-setup) section we have configured and started the Traefik reverse proxy. Now, we will finish the reverse proxy setup on Puddle side.

Since HTTPS is required between ReverseProxy <-> Puddle, make sure the following is set in `/etc/puddle/config.yaml`:

tls:
    certFile: /opt/h2oai/puddle/certs/server/cert.pem
    keyFile: /opt/h2oai/puddle/certs/server/key.pem

Set in `/opt/h2oai/puddle/data/traefik/puddle.yaml` a field `http.services.puddle.loadBalancer.servers` to contain an element `- url: https://localhost:8081`:

http:
    routers:
        puddle:
            rule: "PathPrefix(\`/\`, \`/api-token/\`) || Path(\`/api-keys\`)"
            service: puddle
            tls: {}
            priority: 1
    services:
        puddle:
            loadBalancer:
                servers:
                    \- url: https://localhost:8081

Puddle should run on a custom **port**. This port has to match the port in Puddle Default Rule in Traefik config (`/opt/h2oai/puddle/data/traefik/puddle.yaml`). Make sure that port in `connection.port` (in `/etc/puddle/config.yaml`) and the one used in Puddle Default Rule matches. By default the port should be **8081**.

Finally, make sure that `reverseProxy.*` is configured correctly in `/etc/puddle/config.yaml`. By default the reverse proxy should be enabled (`reverseProxy.enabled: true`) and the paths to keys and certificates should match paths from the keys and certificates generated from section [Generate certificates](#generate-certificates) in Reverse Proxy Setup:

reverseProxy:
    enabled: true
    port: 443
    caCertificate: /opt/h2oai/puddle/certs/ca/cert.pem
    caPrivateKey: /opt/h2oai/puddle/certs/ca/key.pem
    clientCertificate: /opt/h2oai/puddle/certs/client/cert.pem
    clientPrivateKey: /opt/h2oai/puddle/certs/client/key.pem

Now the Traefik reverse proxy is configured on Puddle side.

### Configuring Environment Variables[¶](#configuring-environment-variables "Permalink to this headline")

The next step is to to fill in the variables in EnvironmentFile file, which is located at `/etc/puddle/EnvironmentFile`. The EnvironmentFile should contain the following:

\# Should point to dir with config.yaml
PUDDLE\_config\_dir='/etc/puddle/'

\# AzureRM Provider should skip registering the Resource Providers
ARM\_SKIP\_PROVIDER\_REGISTRATION=true

\# Azure related environment variables, please fill-in all values if you use Azure as provider
\# AZURE\_SUBSCRIPTION\_ID='YOUR-SUBSCRIPTION-ID'
\# AZURE\_TENANT\_ID='YOUR-TENANT-ID'
\# AZURE\_CLIENT\_ID='YOUR-CLIENT-ID'
\# AZURE\_CLIENT\_SECRET='YOUR-CLIENT-SECRET'

\# AWS related environment variables
\# Fill-in the following credentials, unless you use IAM role attached to EC2 instance
\# AWS\_ACCESS\_KEY\_ID='YOUR-AWS-ACCESS-KEY-ID'
\# AWS\_SECRET\_ACCESS\_KEY='YOUR-AWS-SECRET-ACCESS-KEY'
\# \[Required\] region is always required when using AWS as provider
\# AWS\_REGION='AWS-REGION'

\# General variables, delete those which are not necessary
\# http\_proxy=http://10.0.0.100:3128
\# https\_proxy=http://10.0.0.100:3128
\# no\_proxy=localhost,127.0.0.1

*   `PUDDLE_config_dir` **directory** where the config.yaml file is present.
    
*   `ARM_SKIP_PROVIDER_REGISTRATION` - AzureRM Provider should skip registering the Resource Providers. This should be left as true.
    
*   `AZURE_SUBSCRIPTION_ID` is the ID of the subscription that should be used. This value can be found in following way:
    
    > *   Search for **Subscriptions**.
    > *   Use the **SUBSCRIPTION ID** of the subscription you want to use.
    
*   `AZURE_TENANT_ID` is ID of tenant that should be used. This value can be found in following way:
    
    > *   Select **Azure Active Directory** blade.
    > *   Select **App registrations (preview)**.
    > *   Select the newly created App registration.
    > *   Use **Directory (tenant) ID**.
    
*   `AZURE_CLIENT_ID` is the Application ID that should be used. This value can be found in following way:
    
    > *   Select **Azure Active Directory** blade.
    > *   Select **App registrations (preview)**.
    > *   Select the newly created App registration.
    > *   Use **Application (client) ID**.
    
*   `AZURE_CLIENT_SECRET` client secret that should be used. This value can be found in following way:
    
    > *   Select the **Azure Active Directory** blade.
    > *   Select **App registrations (preview)**.
    > *   Select the newly created App registration.
    > *   Select **Certificates & Secrets**.
    > *   Click **New client secret**.
    > *   Fill in the form and click **Add**.
    > *   The secret value should be visible. Copy it because after refreshing the page, this value is gone and cannot be restored.
    
*   `AWS_ACCESS_KEY_ID` AWS Access Key Id used by Puddle to access the AWS services.
    
*   `AWS_SECRET_ACCESS_KEY` AWS Secret Access Key used by Puddle to access the AWS services.
    
*   `AWS_REGION` AWS Region used by Puddle to access the AWS services.
    
*   `http_proxy` is the URL of proxy server to be used (if required), for example [http://10.0.0.3:3128](http://10.0.0.3:3128).
    
*   `https_proxy` is the URL of proxy server to be used (if required), for example [http://10.0.0.3:3128](http://10.0.0.3:3128).
    
*   `no_proxy` is the comma-separated list of hosts that should be excluded from proxying, for example localhost,127.0.0.1.
    

Note that you don’t need to enter credentials on GCP by default, since Puddle uses application default credentials (i.e., the VM service account).

### Running Puddle[¶](#running-puddle "Permalink to this headline")

After all of the previous steps are successfully completed, we can now start Puddle. Execute the following command to start the server and web UI:

systemctl start puddle

Puddle is accessible on port 443 if HTTPS is enabled, or on port 80 if HTTP is being used.

### First Steps[¶](#first-steps "Permalink to this headline")

At first, you will have to perform some initialization steps:

1.  Log in to Puddle as the Administrator.
2.  Go to **Administration > Check Updates**.
3.  Either use the update plan from the default URL location, or specify a custom update plan file.
4.  Click **Submit**.
5.  Review the plan and click **Apply**.
6.  Go to **Administration > Images**.
7.  Build all the images you want to use. Please be aware this can take up to 1 hour.

Once the images are built, your Puddle instance is ready.

### Stats Board (Optional)[¶](#stats-board-optional "Permalink to this headline")

The stats board is an optional component. It’s distributed as Python wheel, and it requires Python 3.6. It’s recommended (although not necessary) to run the board inside a virtual environment.

Use the following to install the required dependencies:

apt install gcc libpq-dev python3.6-dev python-virtualenv

yum install epel-release
yum install gcc postgresql-devel python36-devel python-virtualenv

Use the following to create the virtualenv:

mkdir -p /opt/h2oai/puddle/envs
cd /opt/h2oai/puddle/envs
virtualenv -p python3.6 puddle-stats-env

Please make sure that the virtualenv uses the **same name** and is available at the **same path** as in this provided snippet. Otherwise the systemd script used to manage Stats Board will not work.

Use the following to install the stats board. Please note that this command will install dependencies as well:

source /opt/h2oai/puddle/envs/puddle-stats-env/bin/activate
pip install puddle\_stats\_board-<VERSION>-py3-none-any.whl

Use the following to run the stats board:

systemctl start puddle-dashboard

The stats board is running on port 8050 and is accessible from Puddle UI at **http://<PUDDLE\_SERVER\_ADDRESS>/board**. There is a link in the **Administration** menu as well.

[Next](install-gcp.html "GCP Setup Guide") [Previous](install-azure.html "Azure VPC Setup Guide")

* * *

© Copyright 2019-2020 H2O.ai.. Last updated on Mar 03, 2022.

Built with [Sphinx](http://sphinx-doc.org/) using a [theme](https://github.com/snide/sphinx_rtd_theme) provided by [Read the Docs](https://readthedocs.org).

var DOCUMENTATION\_OPTIONS = { URL\_ROOT:'./', VERSION:'1.7.10', COLLAPSE\_INDEX:false, FILE\_SUFFIX:'.html', HAS\_SOURCE: true, SOURCELINK\_SUFFIX: '' }; jQuery(function () { SphinxRtdTheme.StickyNav.enable(); });
