# IaC-AWS-ECS
Infrastructure as Code for creating AWS Elastic Container Service using Cloud Formation.

This deployment contains CloudFormation templates, parameters file and AWS cli command to create a static website and AWS ECS infrastructure. Some templates can be downloaded from available noted sources. I have separated them into multiple templates so they are reusable when creating other services.

These deployment steps can also be run through automatically using DevOps deployment tool such as Azure DevOps to create CloudFormation stack and retrieve outputs to be passed as parameters in next steps.

# Deployment Steps:

1. Deploy 2 VPCs a. Create CloudFormation stack for template: VPC_template.json 

   This template will create:
   1. 2 VPCs
   2. 2 subnets for 2 availability zones for load balancing VPC1
   3. 1 subnet for VPC2
   4. ACL network (Allow all traffic as default)
   5. Route Tables
   6. Internet Gateway
   7. VPC peering (so these 2 VPCs can communicate to each other privately)

   Note: Be mindful of overlapping IP addresses in these 2 VPCs when select CidrBlock for VPC and subnet.

   Example: powershell and AWS cli

   Use this script to create CloudFormation stack and use default parameters

   aws cloudformation create-stack --stack-name create2VPCs --template-body file://VPC_template.json 

   OR

   Use this script to create CloudFormation stack and provide parameters

   aws cloudformation create-stack --stack-name create2VPCs --template-body file://VPC_template.json -parameters ParameterKey= VPC1Name,ParameterValue=Dev1 ParameterKey= VPC2Name,ParameterValue=Dev2
 
2. Deploy a static http front end service

   Option 1: Use S3 and CloudFront to host a static http as it gives benefits of speeding up the delivery of the static content depending on viewer’s location and no required instance management.
   Source: https://github.com/sjevs/cloudformation-s3-static-website-with-cloudfront-and-route-53

   Create CloudFormation stack for template: route53-zone.yaml and s3-staticwebsite-with-cloudfront-and-route-53.yaml

      These templates will create:
      1. Route53 – Hosted Zone
      2. S3 – Allow Public as default
      3. S3 – Backet Policy (Allow all traffic is default)
      4. CloudFront

      Example: powershell and AWS cli

      aws cloudformation create-stack --stack-name CreateRoute53 --template-body file://route53-zone.yaml --parameters ParameterKey=DomainName,ParameterValue=testmycode.com

      aws cloudformation create-stack --stack-name CreateS3CloudFront --template-body file://s3-staticwebsite-with-cloudfront-and-route-53.yaml --parameters ParameterKey=DomainName,ParameterValue=testmycode.com ParameterKey=FullDomainName,ParameterValue=www.testmycode.com

   Then deploy contents to S3 using deployment pipeline.

   Option2: Create an EC2 instance to host a http static content see step 3. Then setup hosting configuration (eg. IIS) on the instance and deploy content using deployment pipeline.
 
3. Deploy ALB for Bastion Instance or an Instance to host a http static

   Sources: 
   * https://github.com/awsdocs/aws-cloudformation-userguide/blob/master/doc_source/aws-resource-elasticloadbalancingv2-targetgroup.md
   * https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/quickref-ec2.html

   Option 1: Create ALB and EC2 instance separately, then add EC2 to the ALB target group.

      * Create CloudFormation stack for template: ALB_template.json

        This template will create:
           1. VPC (if not specify parameter -> VpcId)
           2. Security Group for ALB (if not specify parameter ->  SecurityGroupId), default ingress port is 80 and egress opens to all. Allow all traffics as default. Can add  port 22 for bastion and set to allow traffic from specific IP Address range.
           3. Application Load Balancer (ALB)
           4. ALB Listener with default rule to forward to the target group
           5. ALB Target group without targets
           6. Output – ALB - DNSname
      
      * Deploy EC2 Instance 

        Source: https://github.com/tongueroo/cloudformation-examples/blob/master/templates/singleinstance.yml

        Create CloudFormation stack for template: single-instance.yml

        This template requires an existing EC2 KeyPair and will create:
          1. EC2 Instance
          2. EC2 Instance security group – Can specify IP Address rage that allows to SSH (port 22) to the Instance
          3. Output – Instance ID

      * Add EC2 to the ALB target group

        Example: powershell and AWS cli

        To register instance to the target group: aws elbv2 register-targets --region ap-southeast-2 --target-group-arn $arntagetgroup -targets Id=$id

        To deregister instance to the target group: aws elbv2 deregister-targets --region ap-southeast-2 --target-group-arn $arntagetgroup --targets Id=$id
     
   Option 2: Create ALB and EC2 instance with target group.

     Create CloudFormation stack for template: ALB-EC2_template.json

     This template will create:
     1. VPC (if not specify parameter -> VpcId)
     2. Security Group for ALB (if not specify parameter ->  SecurityGroupId), default ingress port is 80 and egress opens to all. Allow all traffics as default. Can add  port 22 for bastion and set to allow traffic from specific IP Address range.
     3. Application Load Balancer (ALB)
     4. ALB Listener with default rule to forward to the target group
     5. ALB Target group with target to the instance
     6. EC2 Instance using the same security group specified for ALB
     7. Output – ALB - DNSname
 
4. Deploy ECS (Amazon Elastic Container Service) for api services & applications – Assume they are docker images 

   Reference: https://github.com/widdix/aws-cf-templates/tree/master/ecs
 
  * Create CloudFormation stack for template: ALB-ECS_template.json
  
    This template will create:
    1. VPC (if not specify parameter -> VpcId)
    2. Security Group for ALB (if not specify parameter ->  SecurityGroupId), default ingress port is 80 and egress opens to all. Allow all traffics to port 80 as default
    3. Application Load Balancer (ALB)
    4. ALB Listener with rules to forward to target groups
      * Default rule (/) to forward to port 80
      * Rule - if Path is /auth/* to forward to port 8080
      * Rule - if Path is /api/* to forward to port 9001
      * Rule - if Path is / transactions/* to forward to port 500 
    5. ALB Target groups and health checks
      * Port 80 for the base service
      * Port  8080 for the auth service
      * Port  9001 for the api service
      * Port  5000 for the Transactions service 
      * Output – ALB – DSNname
    
  * Create CloudFormation stack for template: ECS_template.json 
  
  This template will create:
  1. VPC and subnets (if not specify parameter -> VpcId), default Availability Zones are "ap-southeast-2a,ap-southeast-2b,ap-southeast-2c"
  2. Security Group for ALB (if not specify parameter ->  SecurityGroupId), default ingress port is 80 and egress opens to all. Allow all traffics to port 80 as default
  3. AutoScaling LaunchConfiguration
  4. EC2 Instance for containers (default parameter -> UserData to add Clustername to the ecs.config file)
 
   Example: powershell and AWS cli

   aws cloudformation create-stack --stack-name create2VPCs --template-body file:// ECS_template.json –parameters file:// ECS_parameters.json
 
5. Deploy mySQL

   Source: https://github.com/awslabs/aws-cloudformationtemplates/blob/master/aws/services/RDS/RDS_MySQL_With_Read_Replica.yaml
 
  * Create CloudFormation stack for template: RDS_MySQL_With_Read_Replica.yaml
  
    This template will create:
    1. Security Group for database access allow port 3306
    2. Database
    3. Replica DB
    4. Output - EC2Platform, DB ConnectionString, Replica ConnectionString

6. Deploy Postgres

   Source: https://github.com/widdix/aws-cf-templates/blob/master/state/rds-postgres.yaml
 
  * Create CloudFormation stack for template: rds-postgres.yaml

    These templates will create: 
    1. Route53 – record set
    2. VPC and subnet can be passed as parameters
    3. Security Group for Bastion and DB access
    4. Postgres instance
       * Can set the db instance to be deployed to multiple Availability Zones for HA by passing parameter – DBMultiAZ as true
       * If DBSnapshotIdentifier passed with value, can set storage encrypted by passing Kms key value and HasKmsKey as true 
       * Storage type is gp2 as default
       * PreferredBackupWindow is set 09:54-10:24 daily as default
       * PreferredMaintenanceWindow is set at sat:07:00-sat:07:30 weekly as default
    5. Output – Template ID, Template Version, Stack Name, Instance Name, DNS Name

7. Configure the applications to communicate with the mySQL and Postgres using details from step 5 and 6.

8. Deploy docker images to ECR (Amazon Elastic Container Registry) using deployment pipeline.

9. Create ECS service and task definition to the ECS cluster and ALB from step 4 for each api services & applications using deployment pipeline Note: specify desire counts in the task definition to scale up/down the ECS instances through auto-scaling configuration.

10. ECS cluster will keep checking health checks of the ALB - task groups and will automatically trying to resolve itself.

# Note: 
Above infrastructure and components are configured with security grousp to allow traffics from other security groups, private IP address range and public IP address range outside VPC or this subscription.
