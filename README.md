# aws-dms-postgres-onprem-to-rds-v1

**Overview of Guide

In this guide, I will be using the AWS Database Migration Service to migrate an EC2-hosted Postgres (or on-prem) to RDS Postgres.

This guide is a walk-thru of how to migrate an on-prem Postgres database to AWS RDS Postgres using the AWS Database Migration Service. I will simulate the on-prem Postgres by deploying it to an EC2 instance with a public IP address. Technically speaking, the connectivity to an EC2 hosted Postgres database is about the same as an on-prem host. 

This guide assumes a non-production Postgres workload and data running in a non-production AWS environment. If you are working in a production environment, please follow best practices for security and any required compliance standards for your project.


