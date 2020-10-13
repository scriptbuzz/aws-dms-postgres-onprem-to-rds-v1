# aws-dms-postgres-onprem-to-rds-v1

**Overview of Guide**

In this guide, I will deply the AWS Database Migration Service to migrate an EC2-hosted Postgres source database (or on-prem) to RDS Postgres target database. I will be working with a Postgres 12 DB so the commands and settings will reflect that. Please ensure that the commands are updated to reflect your version of Postgres. 


This guide assumes a non-production Postgres workload data running in a non-production AWS environment. If you are working in a production environment, please follow best practices for cloud security as well any compliance requirements. To simplify the guide and keep it to a magable size, workloads are provisioned with public endpoints,  no encryption, and using basic authintication. 


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
•	Connect to the EC2 instance using your favorite method. I prefer to use the AWS SSM Remote Session. 
Now that my EC2 instance is up and running and I am at the command prompt, I will install and configure Postgres 12.
* Run the following commands:
```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get -y install postgresql-12
```
* To allow remote connection to my PostgreSQL DB, I will modify the postgresql.conf file. Please update the command path to reflect your Postgres version number:
```bash
sudo vim /etc/postgresql/12/main/postgresql.conf
```

* Locate the line #listen_addresses = ‘localhost’ and change/uncomment to:
```bash
listen_addresses = '*'
```


