# Github Readme : Automate high availability setup in RDS Custom for Oracle 

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

Detailed documentation of this solution is published as blog and available by following the link below.  <<<<. Blog Link to follow >>>>>> 

## Architecture 

![Arch](/Images/Architecture.png)



## Prerequisites

The CloudFormation script doesn’t create or configure any of the following resources, all of which are required for RDS Custom: 

* VPC
* IAM roles
* Instance profiles
* Amazon S3 buckets
* KMS keys

Review Prerequisites for creating an RDS Custom for Oracle instance in the RDS documentation and complete all preinstallation steps.

Before running the CloudFormation script, make sure to meet the following additional prerequisites:

1. The RDS Custom primary DB instance runs Oracle Database 19c and is in an available state.
2. You have an Amazon S3 bucket that includes the Oracle home client zip file. For the Oracle Database 19c Oracle home, the file is LINUX.X64_193000_client.zip. You can download the full client for Oracle Database 19c from https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html or https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html#license-lightbox.
3. If you use different security groups for RDS and the EC2 observer, make sure that the EC2 observer instance security group permits traffic from the RDS Listener port (for example, 1521) and 1140 for the RDS security group.
4. You have met the general requirementsand network requirements for read replica creation. The CloudFormation script creates an RDS Custom for Oracle read replica.
5. The user running the CloudFormation template has the privileges to run it using the console or CLI.


### List of input parameters to be ready with before deploying the stack

* VpcId - This is the exsiting VPC that hosts the primary DB instance.
* SourceDBInstanceIdentifier - Existing primary database instance identifier 
* SubnetIDforObserver - Existing subnet where EC2 observer will be created 
* EC2SecurityGroup - Existing security group for EC2 Observer to connect RDS Custom Primary and Standby instances
* S3Bucket - Existing S3 bucket path which has the Oracle Client
* TargetDBinstanceAZ - AZ for the replica standby instance. We recommend to choose different AZ than the primary
* RDSCustomInstanceProfile -  Existing RDSCustomInstanceProfile for that region
* TargetDBInstanceIdentifier - Replica/standby instance identifier which will be created by the CFN

## Deploying the solution 

To run the CloudFormation template, complete the following steps:

1. Download the YAML file from Gitlab.
2. Sign in to the AWS Management Console and open the CloudFormation console athttps://console.aws.amazon.com/cloudformation/. Choose the same AWS Region where your RDS Custom primary DB instance resides.
3. Choose Create Stack. 
4. In the Create Stack page, do the following:
    a. Choose Template is ready.
    b. In Upload a template file, choose the YAML file that you downloaded in Step 1.
    c. Choose Next.
5. In Specify Stack Details, enter the following details, and then choose Next:

* For Stack name, enter a descriptive name. The following example uses the name format RDSHA-Primary-DB-instance-identifier, as in RDSHA-primarycustom.
* For VpcId, choose the VPC where observer instance and replica will be created. This is the same VPC that contains the primary DB instance.
    
    >> Follow the same format for the remaining values. For Name, enter value.

![Arch](/Images/StackDetails.png)

1. In Configure stack options, leave the current settings, and then choose Next.
2. In the Review stack-name window, validate the settings, and then select I acknowledge that AWS CloudFormation might create IAM resources with custom names.
3. Choose Submit.



CloudFormation creates an RDS Custom for Oracle read replica and sets the default protection mode to maximum availability. The stack includes an Amazon EC2 observer instance configured with Data Guard Fast-Start Failover. Creation usually takes 2-3 hours, depending on the read replica creation time. All other resources take minimal time to create and set up. To monitor progress, choose Resources in the CloudFormation template. 

![Arch](/Images/stackresources.png)

After the stack deploys successfully, you should see output such as the following.

![Arch](/Images/complete.png)

Additionally, you can also verify the documents installed at your AWS Systems Manager in the shared resources section


## Enable Flashback:

This version of the script does not enable flashback as part of the CFN, hence after a data guard fail over from primary to standby, the old primary has to be rebuilt from backup. In order to automatically reinstate the old primary, Flashback database feature should be enabled on both primary and standby prior to a fail over. Follow the below steps to enable Flashback database before testing the fail over. For your testing, if you do not enable flashback every time after a fail over you have to delete the old primary by disabling FSFO and re create or delete both primary and standby and create a new stack.


* Pause RDS Custom for Oracle database automation for primary and standby database and disable FSFO if it has been enabled previously.
* Log in to the underlying RDS Custom EC2 instances on both primary and standby and create a directory using user rdsdb for storing the flashback logs.

```$ mkdir /rdsdbdata/fra
$ ls -ld /rdsdbdata/fra
drwxrwxr-x 2 rdsdb rdsdb 4096 Feb 2 16:40 /rdsdbdata/fra

$ mkdir /rdsdbdata/fra
$ ls -ld /rdsdbdata/fra
drwxrwxr-x 2 rdsdb rdsdb 4096 Feb 2 16:40 /rdsdbdata/fra
```

* Enable flashback database on both primary and replica

Enable the flashback database on both databases while the database is in MOUNT state:

```
SQL> alter system set dg_broker_start=false scope=both;
SQL> shutdown immediate
SQL> startup mount
SQL> alter system set db_recovery_file_dest='/rdsdbdata/fra';
SQL> alter system set db_flashback_retention_target=60;
SQL> alter system set db_recovery_file_dest_size=10G;
SQL> alter database flashback on;
```

## Troubleshooting Tips

### Troubleshooting a stack creation failure

If CloudFormation fails to create the stack, complete the following steps:

1. On the EC2 observer instance, read /home/ec2-user/bootstrap.log. The following output shows a successful EC2 observer instance deployment.

![Arch](/Images/troubleshoot1.png)

2. In CloudWatch, check the Lambda CloudWatch Log group /aws/lambda/RDS-Custom-HA-Automation-AWS_STACKID for errors when running the SSM document. The following example shows a successful execution of the Lambda function, which includes SSM documents.

![Arch](/Images/troubleshooting2.png)

Note: CloudFormation generates AWS::STACKID automatically. You can find this value on the CloudFormation resources page.

3. In the AWS Systems Manager console, choose Documents.

![Arch](/Images/documents.png)

4. Choose owned by me. Then search for RDS-Custom-HA.

![Arch](/Images/documentslist.png)

## Cleaning up your resources

When you clean up, you can either delete individual resources or the entire stack. We recommend that you delete resources individually as needed.

Deleting resources individually

To clean up a specific resource, follow the documented procedure for deleting this resource. Make sure that you are aware of any dependencies. Consider the following guidelines:

* If the EC2 observer instance is active, you can’t delete the IAM role associated with this instance.
* If Fast-Start Failover is turned on, RDS Custom won’t delete the primary or standby DB instance instance. In this case, turn off Fast-Start Failover and the target, and then use the delete-db-instance command.

For example, if the EC2 observer instance is active, you can’t delete the IAM role associated with this instance.

### Deleting the full stack

If you choose to delete the full stack at once, be aware that the deletion policy for CloudFormation retains the EC2 observer and RDS Custom read replica by default. Turn off this setting at your own risk.


## Contributors 
- Sharath Chandra Kampilis 
- Arnab Saha
- Yamuna Palasamudram

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

