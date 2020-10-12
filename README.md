# aws-dms-postgres-onprem-to-rds-v1

**Overview of Guide**

In this guide, I will deply the AWS Database Migration Service to migrate an EC2-hosted Postgres source database (or on-prem) to RDS Postgres target database.

This guide assumes a non-production Postgres workload data running in a non-production AWS environment. If you are working in a production environment, please follow best practices for cloud security as well any compliance requirements. To simplify the guide and keep it to a magable size, workloads are provisioned with public endpoints,  no encryption, and using basic authintication. 


