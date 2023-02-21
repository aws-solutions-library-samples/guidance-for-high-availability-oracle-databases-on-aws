## Automate high availability setup in RDS Custom for Oracle

## Solution Overview 

This solution is intended to help customers automate the manual steps needed to setup a high availability configuration in RDS Custom for Oracle using read replicas. This is NOT a replacement for Managed Multi-AZ for RDS Custom when it becomes available.

This solution includes a CloudFormation template that will take minimum inputs from user and will perform the following :

* Provision a RDS Custom replica which will act as a standby instance 
* Provision an EC2 instance (t3.medium), install Oracle full client and configure it to become a Data Guard observer
* SSM Documents are deployed to perform the following operations.
    * SSM Document1 - To setup the tnsnames.ora for connectivity to the primary and standby databases from EC2 observer.
    * SSM Document2 - To setup the Wallet creation to include the data guard credentials.
    * SSM Document3 - To make configuration changes on the standby and also enable the fast start fail over.


⚠️ Although this is a non invasive script , make sure you test and run in Dev/ Test environment before you run the scrip in Production. 

Detailed documentation of this solution is published as blog and available by following the link below.  <<<<. Blog Link >>>>>> 

## Architecture 

<<Diagram>>

## Prerequisites

The CloudFormation script doesn’t create or configure any of the following resources, all of which are required for RDS Custom: 

* VPC
* IAM roles
* Instance profiles
* Amazon S3 buckets
* KMS keys

Review Prerequisites for creating an RDS Custom for Oracle instance (https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/custom-setup-orcl.html#custom-setup-orcl.review) in the RDS documentation and complete all preinstallation steps.

Before running the CloudFormation script, make sure to meet the following additional prerequisites:

1. The RDS Custom primary DB instance runs Oracle Database 19c and is in an available state.
2. You have an Amazon S3 bucket that includes the Oracle home client zip file. For the Oracle Database 19c Oracle home, the file is LINUX.X64_193000_client.zip. You can download the full client for Oracle Database 19c from https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html or https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html#license-lightbox.
3. If you use different security groups for RDS and the EC2 observer, make sure that the EC2 observer instance security group permits traffic from the RDS Listener port (for example, 1521) and 1140 for the RDS security group.
4. You have met the general requirements (https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/custom-rr.html#custom-rr.limitations)and network requirements (https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/custom-rr.html#custom-rr.network) for read replica creation. The CloudFormation script creates an RDS Custom for Oracle read replica.
5. The user running the CloudFormation template has the privileges to run it using the console or CLI.
6. You know the following pieces of information:

* The VPC ID. This is the VPC that hosts the primary DB instance.
* The observer AZ. We recommend that you choose a different AZ from the primary and standby DB instances.
* The observer subnet. These are the subnet CIDR ranges corresponding to the preceding AZ.
* The name of the S3 bucket that contains the Oracle full client media.
* The primary DB instance identifier.
* The standby DB instance identifier.
* The standby DB instance AZ.
* The instance profile for the standby database. You can use the same instance profile as the primary database.

## Deploying the solution 

To run the CloudFormation template, complete the following steps:

1. Download the YAML file from Gitlab (https://gitlab.aws.dev/kampilis/amazon-rds-custom-oracle-ha-automation).
2. Sign in to the AWS Management Console and open the CloudFormation console athttps://console.aws.amazon.com/cloudformation/. (https://console.aws.amazon.com/cloudformation/) Choose the same AWS Region where your RDS Custom primary DB instance resides.
3. Choose *Create Stack*. 
4. In the Create Stack page, do the following:
    a. Choose *Template is ready*.
    b. In *Upload a template file*, choose the YAML file that you downloaded in Step 1.
    c. Choose *Next*.
5. In *Specify Stack Details*, enter the following details, and then choose *Next*:

* For *Stack name*, enter a descriptive name. The following example uses the name format *RDSHA-Primary-DB-instance-identifier*, as in *RDSHA-primarycustom*.
* For *VpcId*, choose the VPC where observer instance and replica will be created. This is the same VPC that contains the primary DB instance.
    
    >> Follow the same format for the remaining values. For *Name*, enter *value*.

6. In *Configure stack options*, leave the current settings, and then choose *Next*.
7. In the *Review stack-name* window, validate the settings, and then select *I acknowledge that AWS CloudFormation might create IAM resources with custom names*.
8. Choose *Submit.*

<<Diagram>>

CloudFormation creates an RDS Custom for Oracle read replica and sets the default protection mode to maximum availability. The stack includes an Amazon EC2 observer instance configured with Data Guard Fast-Start Failover. Creation usually takes 2-3 hours, depending on the read replica creation time. All other resources take minimal time to create and set up. To monitor progress, choose *Resources* in the CloudFormation template. 

<<Diagram>>

After the stack deploys successfully, you should see output such as the following.

<<Diagram>>

Additionally, you can also verify the documents installed at your AWS Systems Manager in the shared resources section


## Troubleshooting Tips

## Troubleshooting a stack creation failure

If CloudFormation fails to create the stack, complete the following steps:

1. On the EC2 observer instance, read /home/ec2-user/logs/bootstrap.log. The following output shows a successful EC2 observer instance deployment.

<<Diagram>>

2. In CloudWatch, check the Lambda CloudWatch Log group /aws/lambda/RDS-Custom-HA-Automation- (https://us-west-2.console.aws.amazon.com/cloudwatch/home?region=us-west-2#logsV2:log-groups/log-group/$252Faws$252Flambda$252FRDS-Custom-HA-Automation-02fcad36bb8b)AWS_STACKID for errors when running the SSM document. The following example shows a successful execution of the Lambda function, which includes SSM documents.

<<Diagram>>

*Note:* CloudFormation generates AWS::STACKID automatically. You can find this value on the CloudFormation resources page.

3. In the AWS Systems Manager console, choose *Documents*.

<<Diagram>>

4. Choose *owned by me*. Then search for *RDS-Custom-HA*.

<<diagram>>

## Cleaning up your resources

When you clean up, you can either delete individual resources or the entire stack. We recommend that you delete resources individually as needed.

## Deleting resources individually

To clean up a specific resource, follow the documented procedure for deleting this resource. Make sure that you are aware of any dependencies. Consider the following guidelines:

* If the EC2 observer instance is active, you can’t delete the IAM role associated with this instance.
* If Fast-Start Failover is turned on, RDS Custom won’t delete the primary or standby DB instance instance. In this case, turn off Fast-Start Failover and the target, and then use the delete-db-instance command.

For example, if the EC2 observer instance is active, you can’t delete the IAM role associated with this instance.

## Deleting the full stack

If you choose to delete the full stack at once, be aware that the deletion policy for CloudFormation retains the EC2 observer and RDS Custom read replica by default. Turn off this setting at your own risk.


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

