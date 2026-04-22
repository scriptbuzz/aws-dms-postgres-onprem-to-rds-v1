<div align="center">

# Migrating On-Premises PostgreSQL to Amazon RDS
### Using AWS Database Migration Service (DMS)

[![AWS DMS](https://img.shields.io/badge/AWS-Database_Migration_Service-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/dms/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-12-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Amazon RDS](https://img.shields.io/badge/Amazon-RDS-527FFF?style=for-the-badge&logo=amazonrds&logoColor=white)](https://aws.amazon.com/rds/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)
[![EC2](https://img.shields.io/badge/Amazon-EC2-FF9900?style=for-the-badge&logo=amazonec2&logoColor=white)](https://aws.amazon.com/ec2/)

---

*A step-by-step guide to migrating a self-hosted PostgreSQL 12 database to Amazon RDS using AWS DMS — with minimal downtime.*

[Overview](#overview) · [Prerequisites](#prerequisites) · [Architecture](#architecture) · [Guide](#guide-outline) · [References](#references)

</div>

---

## Overview

Database migrations can be complex, especially when moving from self-managed environments to fully managed cloud services. This guide walks through deploying a complete **AWS Database Migration Service (DMS)** pipeline to migrate a self-hosted PostgreSQL database on Amazon EC2 (simulating on-premises) to an Amazon RDS PostgreSQL target.

> **Video walkthrough:** Watch the complete DMS pipeline in action → [youtu.be/crQDJD6Dj7U](https://youtu.be/crQDJD6Dj7U)

> **Note:** This guide targets non-production workloads. Resources are provisioned with public endpoints and basic authentication to keep the walkthrough concise. For production environments, apply appropriate security hardening, VPC peering, SSL/TLS, and IAM-based authentication.

---

## Technology Stack

| Component | Technology | Details |
|-----------|-----------|---------|
| **Source DB** | PostgreSQL 12 on EC2 | Ubuntu 20.04, simulates on-prem |
| **Target DB** | Amazon RDS PostgreSQL 12 | Fully managed, HA-capable |
| **Migration Engine** | AWS Database Migration Service | Minimal-downtime replication |
| **Client Tool** | pgAdmin | GUI for Postgres administration |

---

## Prerequisites

Before starting, make sure you have:

- [ ] An active **AWS account** with IAM permissions for EC2, RDS, and DMS
- [ ] **pgAdmin** installed on your local machine ([download here](https://www.pgadmin.org/))
- [ ] Familiarity with the AWS Management Console
- [ ] A test CSV dataset to migrate (UK Land Registry data used in this guide)

---

## Architecture

<div align="center">
  <img src="assets/mbx-dms-diagram.png" alt="AWS DMS Architecture Diagram" width="85%"/>
  <br/>
  <em>End-to-end migration architecture: EC2-hosted PostgreSQL → AWS DMS → Amazon RDS PostgreSQL</em>
</div>

---

## Guide Outline

This guide is divided into the following sections:

1. [Deploy an EC2-Hosted PostgreSQL DB](#1-deploy-an-ec2-hosted-postgresql-db)
2. [Provision the Target Amazon RDS PostgreSQL DB](#2-provision-the-target-amazon-rds-postgresql-db)
3. [Set Up pgAdmin](#3-set-up-pgadmin)
4. [Load Test Data Into the Source DB](#4-load-test-data-into-the-source-db)
5. [Provision AWS Database Migration Service](#5-provision-aws-database-migration-service)
   - [Create a Replication Instance](#51-create-a-replication-instance)
   - [Create & Test the Source EC2 Endpoint](#52-create--test-the-source-ec2-endpoint)
   - [Create & Test the Target RDS Endpoint](#53-create--test-the-target-rds-endpoint)
   - [Create the Migration Task](#54-create-the-migration-task)
6. [Resourcing Considerations](#6-resourcing-considerations)

---

## 1. Deploy an EC2-Hosted PostgreSQL DB

**Launch the EC2 Instance**

1. Log in to the **AWS Management Console** and set your AWS Region.
2. Navigate to **EC2 → Launch Instance**.
3. Select the **Ubuntu 20.04 LTS** AMI (Amazon-managed).
4. Choose instance type: `m5a.large`.
5. Use the **default VPC**, select a subnet and Availability Zone.
6. **Enable Auto-assign Public IP** — then attach an Elastic IP after launch to retain a stable address.
7. *(Optional)* Attach an IAM SSM role to allow Session Manager access (no SSH key needed).
8. Allocate **20 GB** of storage.
9. Add a tag to the instance (e.g., `Name: postgres-source`).
10. Attach a Security Group with:
    - Inbound: `TCP 5432` (PostgreSQL)
    - Inbound: `TCP 22` (SSH, if needed)
11. Launch and wait for **2/2 status checks**.
12. Attach an **Elastic IP** and note it — you'll reference it in DMS endpoint configuration.

**Install PostgreSQL 12**

Connect to the EC2 instance (via SSM or SSH) and run:

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
sudo apt-get update
sudo apt-get -y install postgresql-12
```

**Configure Remote Access**

Allow remote connections by updating `postgresql.conf`:

```bash
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/g" /etc/postgresql/12/main/postgresql.conf
```

Append authentication rules to `pg_hba.conf`:

```bash
echo "local   all    postgres       md5"                  | sudo tee -a /etc/postgresql/12/main/pg_hba.conf
echo "host    all    all            0.0.0.0/0    md5"     | sudo tee -a /etc/postgresql/12/main/pg_hba.conf
echo "host    all    all            ::/0         md5"     | sudo tee -a /etc/postgresql/12/main/pg_hba.conf
```

**Set the postgres User Password**

```bash
sudo -u postgres psql template1
```

At the SQL prompt:

```sql
ALTER USER postgres WITH ENCRYPTED PASSWORD 'your_password';
\q
```

**Restart PostgreSQL** to apply changes:

```bash
sudo systemctl restart postgresql.service
```

---

## 2. Provision the Target Amazon RDS PostgreSQL DB

In the **RDS Dashboard**, use the following settings as a baseline:

| Setting | Value |
|---------|-------|
| Engine | PostgreSQL 12 |
| Template | Dev/Test |
| Instance class | `db.m5.large` |
| Storage | 40 GB (autoscaling disabled) |
| Multi-AZ | Disabled (no standby) |
| Public access | Yes |
| Port | 5432 |
| Authentication | Password |

1. Log in → **RDS Dashboard → Create database**.
2. Select **PostgreSQL** → Template: **Dev/Test**.
3. Set a DB identifier, master username (`postgres`), and a strong password.
4. Configure instance size, storage, VPC, and subnet group as above.
5. Set **Public access: Yes** and attach a Security Group with inbound `TCP 5432`.
6. Set an **initial database name** under Database options.
7. Click **Create database** and wait for status: **Available**.
8. Use **View credentials details** to copy your endpoint and credentials.

---

## 3. Set Up pgAdmin

**Required information before proceeding:**

- Public IP of the EC2 source PostgreSQL instance
- RDS endpoint for the target PostgreSQL DB
- Credentials for both databases
- Security groups must allow inbound `TCP 5432` for both DBs

**Steps:**

1. Download and install [pgAdmin](https://www.pgadmin.org/) for your OS.
2. Add a **New Server** for the EC2 source DB and verify connectivity.
3. Add a **New Server** for the RDS target DB and verify connectivity.

<div align="center">
  <img src="assets/mbx-dms-pgadmin-create-server.png" alt="pgAdmin Create Server Dialog" width="70%"/>
  <br/><em>Creating a new server connection in pgAdmin</em>
</div>

<br/>

<div align="center">
  <img src="assets/mbx-dms-pgadmin-connect.png" alt="pgAdmin Connection Success" width="70%"/>
  <br/><em>Successful connection to the PostgreSQL database</em>
</div>

---

## 4. Load Test Data Into the Source DB

This guide uses the **UK Land Registry Price Paid** dataset. You can find PostgreSQL sample datasets at the [PostgreSQL wiki](https://wiki.postgresql.org/wiki/Sample_Databases).

**Create the database and table structure:**

```bash
sudo -u postgres psql template1
```

```sql
-- List databases
\l

-- Create and switch to the test database
CREATE DATABASE mbxDB;
\c mbxDB

-- Create the target table
CREATE TABLE land_registry_price_paid_uk (
  transaction      uuid,
  price            numeric,
  transfer_date    date,
  postcode         text,
  property_type    char(1),
  newly_built      boolean,
  duration         char(1),
  paon             text,
  saon             text,
  street           text,
  locality         text,
  city             text,
  district         text,
  county           text,
  ppd_category_type char(1),
  record_status    char(1)
);

\q
```

**Import the CSV using pgAdmin:**

1. In the Object Browser, expand `mbxDB` → `Tables`.
2. Right-click `land_registry_price_paid_uk` → **Import/Export Data**.
3. Switch to **Import**, select your downloaded CSV file.
4. Click **OK** and wait for the success message.
5. Query the table to verify the imported rows.

---

## 5. Provision AWS Database Migration Service

### 5.1 Create a Replication Instance

1. Open the **DMS Dashboard** → **Replication instances** → **Create replication instance**.
2. Configure:
   - **Name:** assign a meaningful name
   - **Instance class:** `dms.t2.medium`
   - **Engine version:** latest stable (e.g., 3.4.x)
   - **Storage:** 50 GiB
   - **VPC:** Default
   - **Multi-AZ:** Disabled
   - **Publicly accessible:** Yes

<div align="center">
  <img src="assets/mbx-dms-rep-instance01.png" alt="DMS Replication Instance Configuration Part 1" width="70%"/>
</div>
<br/>
<div align="center">
  <img src="assets/mbx-dms-rep-instance02.png" alt="DMS Replication Instance Configuration Part 2" width="70%"/>
</div>

3. Accept defaults for maintenance and advanced settings.
4. Click **Create** and wait for status: **Available**.

---

### 5.2 Create & Test the Source EC2 Endpoint

1. **DMS Dashboard → Endpoints → Create endpoint → Source endpoint**
2. Fill in:

   | Field | Value |
   |-------|-------|
   | Endpoint identifier | `mbx-postgres-ec2-source` |
   | Source engine | PostgreSQL |
   | Server name | EC2 Elastic IP address |
   | Port | `5432` |
   | Username | `postgres` |
   | Password | your EC2 postgres password |
   | Database name | `mbxDB` |

3. Expand **Test endpoint connection**, select your replication instance, and click **Run test**.
4. Wait for status: **Successful**. If it fails, check your Security Group inbound rules for port 5432.

<div align="center">
  <img src="assets/mbx-dms-endpoint04.png" alt="DMS EC2 Source Endpoint" width="70%"/>
</div>
<br/>
<div align="center">
  <img src="assets/mbx-dms-endpoint03.png" alt="DMS EC2 Endpoint Connection Test" width="70%"/>
</div>

---

### 5.3 Create & Test the Target RDS Endpoint

1. **DMS Dashboard → Endpoints → Create endpoint → Target endpoint**
2. Fill in:

   | Field | Value |
   |-------|-------|
   | Endpoint identifier | `mbx-target-postgres-rds` |
   | Select RDS DB instance | ✅ Checked |
   | RDS instance | Select from dropdown |
   | Password | your RDS master password |

3. Expand **Test endpoint connection**, select your replication instance, and click **Run test**.
4. Wait for status: **Successful**.

<div align="center">
  <img src="assets/mbx-dms-endpoint02.png" alt="DMS RDS Target Endpoint" width="70%"/>
</div>
<br/>
<div align="center">
  <img src="assets/mbx-dms-endpoint01.png" alt="DMS RDS Endpoint Connection Test" width="70%"/>
</div>

---

### 5.4 Create the Migration Task

1. **DMS Dashboard → Database migration tasks → Create task**
2. Configure:

   | Field | Value |
   |-------|-------|
   | Task identifier | `mbx-postgres-migration-task` |
   | Replication instance | your replication instance |
   | Source endpoint | `mbx-postgres-ec2-source` |
   | Target endpoint | `mbx-target-postgres-rds` |
   | Migration type | Migrate existing data |
   | Editing mode (task settings) | Wizard |
   | Target table prep | Drop tables on target |
   | LOB mode | Limited LOB mode |
   | Max LOB size | 32 KB |
   | CloudWatch logs | ✅ Enable |

3. Under **Table mappings → Selection rules**:
   - Schema name: `%`
   - Table name: `%`
4. Click **Create task**.
5. If the task doesn't start automatically: **Actions → Restart/Resume**.

<div align="center">
  <img src="assets/mbx-dms-mapping-rules.png" alt="DMS Table Mapping Rules" width="70%"/>
</div>

6. In the **Summary panel**, wait for status: **Load complete**.

<div align="center">
  <img src="assets/mbx-dms-rep-instance-tasks.png" alt="DMS Task Load Complete" width="70%"/>
</div>

7. In the **Table statistics panel**, verify the row count (e.g., `67,788` rows for the sample dataset).

<div align="center">
  <img src="assets/mbx-dms-rep-instance-migration-stats.png" alt="DMS Migration Statistics" width="70%"/>
</div>

8. Connect to your **RDS target DB** via pgAdmin and confirm the data has migrated successfully.

> **Video — Final Results:** See the migrated data in RDS → [youtu.be/QHKhyzPUPaU](https://youtu.be/QHKhyzPUPaU)

---

## 6. Resourcing Considerations

For **production** database migrations, involve the right stakeholders early:

| Role | Responsibility |
|------|---------------|
| **Postgres DBA** | Authentication, performance tuning, replication validation |
| **Domain SME** | Identify which data to migrate, define acceptance criteria |
| **Cloud Architect** | Network topology, security, cost optimization |
| **Project Owner** | SLAs, migration window, rollback plan |

Key questions to answer before any production migration:

- Which schemas/tables need to be migrated?
- What data transformations or type mappings are required?
- What are the SLA requirements (RTO/RPO)?
- What acceptance tests confirm a successful migration?

---

## Summary

By completing this guide you have:

- ✅ Deployed a self-hosted PostgreSQL 12 database on EC2
- ✅ Provisioned an Amazon RDS PostgreSQL 12 target database
- ✅ Loaded a real-world test dataset using pgAdmin
- ✅ Created a DMS Replication Instance, Source Endpoint, and Target Endpoint
- ✅ Successfully migrated data with zero manual SQL scripting

AWS DMS abstracts the heavy lifting of data replication, enabling seamless transition from unmanaged to fully managed database environments with minimal downtime.

---

## References

- [AWS Database Migration Service (DMS)](https://aws.amazon.com/dms/)
- [Install PostgreSQL on Ubuntu](https://ubuntu.com/server/docs/databases-postgresql)
- [PostgreSQL Downloads (Linux/Ubuntu)](https://www.postgresql.org/download/linux/ubuntu/)
- [Deploy an Amazon RDS PostgreSQL DB](https://aws.amazon.com/getting-started/tutorials/create-connect-postgresql-db/)
- [pgAdmin — PostgreSQL Tools](https://www.pgadmin.org/)
- [UK Land Registry Price Paid Data](https://www.gov.uk/government/statistical-data-sets/price-paid-data-downloads)

---

<div align="center">

Made with care by [Mike Bitar](https://github.com/mikebitar) &nbsp;·&nbsp; MIT License

</div>
