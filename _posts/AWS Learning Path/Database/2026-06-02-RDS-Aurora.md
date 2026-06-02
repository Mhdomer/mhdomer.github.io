---
layout: post
title: AWS RDS and Aurora — Managed Relational Databases, Multi-AZ, and Read Replicas
date: 2026-06-02T10:00:00
categories:
  - AWS Learning Path
  - Database
tags:
  - aws
  - rds
  - aurora
  - database
  - cloud
author: muhammed
description: A full walkthrough of AWS RDS and Aurora — supported engines, Multi-AZ, read replicas, automated backups, encryption, RDS Proxy, and Aurora Serverless
toc: true
pin: false
math: false
mermaid: false
image: /assets/notes-img/RDSnarora.png
Link: "[[]]"
Link1:
img:
---

## What is RDS?

**Amazon RDS (Relational Database Service)** is a managed service that makes it easier to set up, operate, and scale relational databases in the cloud.
AWS handles the undifferentiated heavy lifting: hardware provisioning, OS patching, database software updates, backups, and failover.
You focus on your schema and queries — not on database administration.

### Supported Engines

| Engine | Notes |
|---|---|
| **MySQL** | Most popular open-source RDBMS |
| **PostgreSQL** | Advanced features, JSON support, extensible |
| **MariaDB** | MySQL fork, community-driven |
| **Oracle** | Enterprise; bring-your-own-license or license-included |
| **SQL Server** | Microsoft; bring-your-own-license or license-included |
| **Amazon Aurora** | AWS-built, MySQL and PostgreSQL compatible — covered separately below |

### RDS vs Self-Managed on EC2

| | RDS | EC2 + Database |
|---|---|---|
| OS access | No | Yes (full SSH) |
| DB software updates | AWS | You |
| Backups | Automated | You |
| Multi-AZ failover | One click | Complex to implement |
| Cost | Higher per unit | Lower per unit, more work |
| Use when | Standard HA requirements | Custom DB version, special OS settings |

---

## DB Instances

A DB instance is a database environment running in the cloud.
You choose the instance class (CPU + RAM) and storage independently.

### Instance Classes

| Class | Purpose |
|---|---|
| **db.t3 / db.t4g** | Burstable — dev/test, low-traffic |
| **db.m6g / db.m7g** | General purpose — balanced CPU/memory |
| **db.r6g / db.r7g** | Memory optimised — large databases, high connection counts |
| **db.x2g** | Extreme memory — SAP HANA, very large in-memory workloads |

Graviton (g-suffix) instances offer ~20% better price-performance.

### Storage Types

| Type | IOPS | Use Case |
|---|---|---|
| **gp3** | Up to 64,000 | General purpose (default) |
| **io1** | Up to 256,000 | I/O-intensive databases |
| **Magnetic** | Low | Legacy — do not use for new databases |

Storage autoscaling is available — RDS can automatically increase storage when it runs low, up to a configured maximum.

```bash
# Create an RDS PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier prod-postgres \
  --db-instance-class db.r7g.large \
  --engine postgres \
  --engine-version 16.2 \
  --master-username admin \
  --master-user-password "$(aws secretsmanager get-secret-value --secret-id db-master-password --query SecretString --output text)" \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --db-subnet-group-name prod-db-subnet-group \
  --vpc-security-group-ids sg-db \
  --multi-az \
  --backup-retention-period 7 \
  --no-publicly-accessible

# List instances
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address]' \
  --output table
```

> 📸 **SCREENSHOT:** RDS → Create Database.
> Show the engine selection (PostgreSQL), template (Production), DB instance class, and the Multi-AZ deployment option enabled.

---

## DB Subnet Groups

A **DB subnet group** is a collection of subnets in different AZs that RDS uses to place your database instances.
Always create a DB subnet group across at least 2 AZs — required for Multi-AZ.
Put database subnets in private subnets with no internet gateway route.

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name prod-db-subnet-group \
  --db-subnet-group-description "Production database subnets" \
  --subnet-ids subnet-db-1a subnet-db-1b subnet-db-1c
```

---

## Multi-AZ Deployments

**Multi-AZ** keeps a synchronous standby replica of your database in a different Availability Zone.
If the primary instance fails (hardware failure, AZ outage), RDS automatically fails over to the standby — typically in under 2 minutes.
The failover is automatic; your application reconnects to the same DNS endpoint.

```
Primary (eu-west-1a) ──synchronous replication──► Standby (eu-west-1b)
        │
        └── Single DNS endpoint (always points to primary)
```

**Multi-AZ is for availability (HA), not performance.**
The standby does not serve read traffic — it only exists to take over when the primary fails.

> 📸 **SCREENSHOT:** RDS → DB instance → Configuration tab.
> Show "Multi-AZ" set to "Yes" and the Availability Zone of the primary and the secondary AZ listed.

```bash
# Convert a single-AZ instance to Multi-AZ (applied immediately or at next maintenance window)
aws rds modify-db-instance \
  --db-instance-identifier prod-postgres \
  --multi-az \
  --apply-immediately

# Trigger a failover (for testing)
aws rds reboot-db-instance \
  --db-instance-identifier prod-postgres \
  --force-failover
```

---

## Read Replicas

**Read replicas** are copies of the primary database that handle read traffic — they use asynchronous replication.
Use them to offload read queries from the primary (SELECT-heavy reporting, analytics).

| Property | Value |
|---|---|
| Max read replicas | 5 per source (MySQL/MariaDB/PostgreSQL) |
| Replication | Asynchronous — small lag behind primary |
| Promotion | Replica can be promoted to standalone DB (breaks replication) |
| Cross-region | Supported — create replicas in different regions |
| Endpoint | Each replica has its own DNS endpoint |

```bash
# Create a read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-postgres-replica-1 \
  --source-db-instance-identifier prod-postgres \
  --db-instance-class db.r7g.large \
  --availability-zone eu-west-1b

# Cross-region read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-postgres-us-replica \
  --source-db-instance-identifier prod-postgres \
  --region us-east-1

# Promote a replica to standalone (breaks replication permanently)
aws rds promote-read-replica \
  --db-instance-identifier prod-postgres-replica-1
```

### Multi-AZ vs Read Replicas

| | Multi-AZ | Read Replica |
|---|---|---|
| **Purpose** | High availability / failover | Read scaling / offload reads |
| **Replication** | Synchronous | Asynchronous |
| **Serves traffic** | Standby does NOT serve reads | Yes — serves read traffic |
| **Failover** | Automatic | Manual promotion |

---

## Automated Backups and Snapshots

### Automated Backups

RDS takes daily automated backups and logs transaction logs continuously.
This enables **Point-in-Time Recovery (PITR)** — restore to any second within the retention window.
Retention period: 0 to 35 days (0 disables automated backups).

```bash
# Restore to a point in time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-postgres \
  --target-db-instance-identifier prod-postgres-restored \
  --restore-time 2026-05-31T14:30:00Z
```

### Manual Snapshots

Manual snapshots are user-initiated and persist until you delete them — they don't expire.
Use for pre-migration snapshots, quarterly archives, or before major schema changes.

```bash
# Create a snapshot
aws rds create-db-snapshot \
  --db-instance-identifier prod-postgres \
  --db-snapshot-identifier prod-postgres-before-migration

# List snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier prod-postgres \
  --output table

# Restore from snapshot (creates a new instance)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-postgres-v2 \
  --db-snapshot-identifier prod-postgres-before-migration

# Copy snapshot to another region
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:eu-west-1:123456789012:snapshot:prod-postgres-snap \
  --target-db-snapshot-identifier prod-postgres-snap-us \
  --region us-east-1
```

---

## Encryption

RDS supports encryption at rest using **AWS KMS**.
Encryption must be enabled at creation time — you cannot encrypt an existing unencrypted instance directly.

To encrypt an existing unencrypted instance:
1. Take a snapshot of the unencrypted instance
2. Copy the snapshot with encryption enabled
3. Restore a new instance from the encrypted snapshot
4. Switch your application to the new endpoint

All data in encrypted instances is encrypted — DB storage, automated backups, read replicas, and snapshots.

In transit: RDS supports SSL/TLS connections.
For PostgreSQL, set `rds.force_ssl=1` in the parameter group to require SSL for all connections.

---

## Parameter Groups and Option Groups

**Parameter groups** contain engine configuration settings — the equivalent of `my.cnf` or `postgresql.conf`.
Create a custom parameter group when you need non-default settings (e.g. `max_connections`, `shared_buffers`).

```bash
# Create a custom parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name prod-postgres-params \
  --db-parameter-group-family postgres16 \
  --description "Production PostgreSQL parameters"

# Modify a parameter
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-postgres-params \
  --parameters 'ParameterName=max_connections,ParameterValue=500,ApplyMethod=pending-reboot'
```

**Option groups** add optional features to the DB engine — e.g. Oracle TDE, SQL Server Transparent Data Encryption.
Most PostgreSQL and MySQL workloads don't need a custom option group.

---

## RDS Proxy

**RDS Proxy** is a managed connection pool that sits between your application and RDS.
It maintains a pool of established database connections and multiplexes thousands of application connections onto fewer database connections.

Use it when:
- Your application opens and closes database connections frequently (Lambda, serverless)
- You need to reduce connection overhead during traffic spikes
- You want automatic failover handling — the proxy keeps connections alive during Multi-AZ failover

```bash
# Create an RDS Proxy
aws rds create-db-proxy \
  --db-proxy-name prod-postgres-proxy \
  --engine-family POSTGRESQL \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:...:secret:db-creds","IAMAuth":"REQUIRED"}]' \
  --role-arn arn:aws:iam::123456789012:role/rds-proxy-role \
  --vpc-subnet-ids subnet-private-1a subnet-private-1b \
  --vpc-security-group-ids sg-rds-proxy
```

> 📸 **SCREENSHOT:** RDS → Proxies → Create Proxy.
> Show the target database selected, the Secrets Manager secret chosen for credentials, and the IAM authentication toggle enabled.

---

## Aurora

**Amazon Aurora** is AWS's own cloud-native relational database engine.
It is fully compatible with **MySQL** and **PostgreSQL** — most applications work without code changes.
Aurora outperforms standard MySQL by up to 5× and PostgreSQL by up to 3×, at a lower cost than commercial databases.

### Aurora Architecture

Aurora separates compute from storage.
Storage is a distributed, fault-tolerant, self-healing system that automatically replicates 6 copies of your data across 3 AZs.
The storage volume grows automatically in 10 GB increments as your data grows — up to 128 TB.

```
Aurora Writer (primary) ──── reads/writes ────► Shared Distributed Storage (6 copies, 3 AZs)
Aurora Reader 1         ──── reads ──────────►
Aurora Reader 2         ──── reads ──────────►  (up to 15 read replicas)
```

Because all readers and the writer share the same storage, read replicas have zero replication lag.
Promotion of a reader to writer during failover takes under 30 seconds.

> 📸 **SCREENSHOT:** RDS → Databases → select Aurora cluster → Connectivity & security tab.
> Show the Cluster endpoint (writer) and Reader endpoint, and the list of instances showing which is the writer and which are readers.

### Aurora vs RDS

| | Aurora | RDS |
|---|---|---|
| **Replication** | Shared storage, zero lag | Asynchronous log shipping |
| **Read replicas** | Up to 15 | Up to 5 |
| **Failover time** | < 30 seconds | < 120 seconds |
| **Storage** | Auto-grows to 128 TB | Must provision size |
| **Cost** | Higher compute cost | Lower compute cost |
| **Backup** | Continuous to S3 | Daily snapshots + transaction logs |

### Aurora Serverless v2

Aurora Serverless v2 automatically scales compute capacity in fine-grained increments based on actual load.
Scales from 0.5 ACUs (Aurora Capacity Units) to 128 ACUs — each ACU is approximately 2 GB RAM.
Ideal for variable workloads, dev/test environments, and applications with unpredictable traffic.

```bash
# Create an Aurora Serverless v2 cluster
aws rds create-db-cluster \
  --db-cluster-identifier prod-aurora-serverless \
  --engine aurora-postgresql \
  --engine-version 16.1 \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
  --master-username admin \
  --master-user-password secret \
  --db-subnet-group-name prod-db-subnet-group \
  --vpc-security-group-ids sg-db
```

### Aurora Global Database

**Aurora Global Database** replicates your Aurora cluster to up to 5 secondary regions with **under 1 second replication lag**.
Use for disaster recovery with near-zero RPO, or to serve read traffic from multiple regions with low latency.

In a DR scenario, you can promote a secondary region to the primary in under 1 minute.

---

## Quick Reference

```bash
# RDS instances
aws rds describe-db-instances --output table
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceClass,DBInstanceStatus,Endpoint.Address,MultiAZ]' \
  --output table

# Start / stop (dev instances — not Multi-AZ)
aws rds stop-db-instance --db-instance-identifier dev-postgres
aws rds start-db-instance --db-instance-identifier dev-postgres

# Snapshots
aws rds describe-db-snapshots --output table
aws rds delete-db-snapshot --db-snapshot-identifier old-snapshot

# Aurora clusters
aws rds describe-db-clusters --output table

# Events (failover, backup, maintenance)
aws rds describe-events \
  --source-type db-instance \
  --source-identifier prod-postgres \
  --duration 10080  # last 7 days in minutes

# Performance Insights (identify slow queries)
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db-abc123 \
  --metric-queries '[{"Metric":"db.load.avg"}]' \
  --start-time $(date -d '1 hour ago' --iso-8601=seconds) \
  --end-time $(date --iso-8601=seconds) \
  --period-in-seconds 60
```
