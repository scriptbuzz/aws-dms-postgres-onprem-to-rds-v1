# aws-dms-postgres-onprem-to-rds-v1

**Overview of Guide**

In this guide, I will deply the AWS Database Migration Service to migrate an EC2-hosted Postgres source database (or on-prem) to RDS Postgres target database.

This guide assumes a non-production Postgres workload data running in a non-production AWS environment. If you are working in a production environment, please follow best practices for cloud security as well any compliance requirements. To simplify the guide and keep it to a magable size, workloads are provisioned with public endpoints,  no encryption, and using basic authintication. 

I will deploy a Postgres 12 DB so the command line options will reflect that. Ensure that the commands are updated to reflect your version of Postgres. 

***Guide Outline***
This guide is made up of the following sections:
* Deploy an EC2 hosted Postgres DB
* Deploy an RDS Postgres DB
* Install pgAdmin, a popular GUI client for Postgres DB
* Import test data to the EC2 Postgres DB
* Deploy the Database Migration Service (DMS)
  * Create the DMS Replication Instance
  * Create and Test The DMS Source Endpoint
  * Create and Test The DMS Target Endpoint
  * Create a DMS Migration Task
  * Run the DMS Migration Task
  * Verify table migration from source DB to target DB. 
  
  
  
***Deploy an EC2-hosted Postgres DB***

* Login to the AWS Management Console 
* Set your AWS Region
* From EC2 Dashboard: 
* Launch a new EC2 instance using the Amazon-managed Ubuntu 20.x AMI
* Select EC2 Instance Type m5a.large (or the type that works best for you)
* Select the default VPC
* Select subnet/AZ
* Enable Assign Public IP. Later, attach an EIP so you don't end up with a different public IP address everytime you restart your EC2 instance. 
* Optional: Attach IAM SSM Role (to allow remote access into EC2 without the need for SSH)
* For storage, assign 20 GB for good measures. This is more than enough for the test dataset
* Assign a useful Name tag to the EC2 instance
* Select Security Group with inbound rule for the Postgres port. Typically, it is port 5432. If you will be using SSH, also open port 22 
•	Review your configuration and Launch the EC2 instance
•	Wait for EC2 instance until it's "Running" with a status check of 2/2
•	Attach an Elastic Public IP Address. Note the public IP address of your EC2 instance. You will use this IP when configuring your server access as well as the DMS endpoints. 
•	Connect to the EC2 instance using your favorite method. I prefer the SSM Remote Session. 


