---
layout: post
title: "Storage type overview"
date: 2024-04-01 06:55:00 +0000
categories: [storage]
tags: []
author: alphajerboa
license: "CC BY 4.0"
license_url: "https://creativecommons.org/licenses/by/4.0/"
excerpt: "Lost in all the storage options, check this out"
---

Lost in all the storage options? Here's a quick overview of the different storage types, their common use cases, and the most popular tools for each.

## Block Storage

**Usage**: Raw storage that appears as individual drives to operating systems. Used for databases, file systems, boot volumes, and any application requiring high-performance, low-latency access.

**Common Tools**:
- Amazon EBS (Elastic Block Store)
- Azure Disk Storage
- Google Persistent Disk
- NetApp ONTAP
- Dell EMC PowerStore
- HPE 3PAR

## File Storage

**Usage**: Hierarchical storage with directories and subdirectories. Used for shared file access, content repositories, media storage, backup and archival, and applications requiring POSIX file system semantics.

**Common Tools**:
- Amazon EFS (Elastic File System)
- Azure Files
- Google Filestore
- NetApp Cloud Volumes
- IBM Spectrum Scale
- Lustre (for HPC)

## Object Storage

**Usage**: Scalable storage for unstructured data accessed via REST APIs. Used for web applications, content distribution, backup and archival, data lakes, and static website hosting.

**Common Tools**:
- Amazon S3
- Azure Blob Storage
- Google Cloud Storage
- MinIO
- OpenStack Swift
- Wasabi
- Backblaze B2

## Key-Value Storage

**Usage**: Simple database model storing data as key-value pairs. Used for session management, caching, user preferences, shopping carts, and real-time recommendations.

**Common Tools**:
- Redis
- Amazon DynamoDB
- Azure Cosmos DB
- Riak
- Apache Cassandra (wide-column, but often used as key-value)
- Memcached
- Hazelcast

## Relational/SQL Storage

**Usage**: Structured data with defined relationships and ACID compliance. Used for transactional applications, financial systems, CRM, ERP, and applications requiring complex queries and data integrity.

**Common Tools**:
- PostgreSQL
- MySQL
- Microsoft SQL Server
- Oracle Database
- Amazon RDS
- Azure SQL Database
- Google Cloud SQL

## Document Storage (NoSQL)

**Usage**: Storing semi-structured data as documents (JSON, XML, BSON). Used for content management, catalogs, user profiles, and applications with evolving schemas.

**Common Tools**:
- MongoDB
- Amazon DocumentDB
- Azure Cosmos DB
- CouchDB
- Couchbase
- Firebase Firestore

## Column-Family/Wide-Column Storage

**Usage**: Storing data in column families rather than rows. Used for time-series data, IoT applications, logging, analytics, and applications requiring high write throughput.

**Common Tools**:
- Apache Cassandra
- Apache HBase
- Amazon DynamoDB
- Azure Cosmos DB
- Google Bigtable
- ScyllaDB

## Graph Storage

**Usage**: Storing data as nodes and relationships. Used for social networks, recommendation engines, fraud detection, network analysis, and knowledge graphs.

**Common Tools**:
- Neo4j
- Amazon Neptune
- Azure Cosmos DB (Gremlin API)
- ArangoDB
- TigerGraph
- Apache TinkerPop

## Time-Series Storage

**Usage**: Optimized for time-stamped data. Used for monitoring, IoT sensor data, financial market data, application metrics, and log analysis.

**Common Tools**:
- InfluxDB
- Amazon Timestream
- TimescaleDB
- OpenTSDB
- Prometheus
- Grafana (with various backends)

## In-Memory Storage

**Usage**: Storing data in RAM for ultra-fast access. Used for caching, real-time analytics, session storage, and high-frequency trading applications.

**Common Tools**:
- Redis
- Memcached
- Apache Ignite
- Hazelcast
- SAP HANA
- VoltDB

## Network Storage

**Usage**: Storage accessed over a network connection, allowing multiple clients to share storage resources. Provides centralized storage management, backup, and redundancy.

### Network Attached Storage (NAS)
**Usage**: File-level storage accessible over a network, typically using protocols like NFS, SMB/CIFS, or AFP. Used for file sharing, media streaming, backup, and collaborative workspaces.

**Common Tools**:
- Synology DiskStation
- QNAP NAS
- NetApp FAS/AFF Series
- Dell EMC Isilon
- Drobo
- FreeNAS/TrueNAS

### Storage Area Network (SAN)
**Usage**: Block-level storage accessed over high-speed networks, typically using protocols like Fibre Channel, iSCSI, or FCoE. Used for enterprise databases, virtualization, and high-performance applications requiring low latency.

**Common Tools**:
- Dell EMC PowerMax/Unity
- HPE 3PAR/Nimble
- NetApp FAS/AFF (iSCSI/FC)
- Pure Storage FlashArray
- IBM FlashSystem
- Brocade/Cisco SAN switches

### Distributed/Clustered Storage
**Usage**: Storage distributed across multiple nodes in a network, providing scalability and fault tolerance. Used for big data, cloud storage, and applications requiring massive scale and high availability.

**Common Tools**:
- Ceph
- GlusterFS
- Lustre
- HDFS (Hadoop Distributed File System)
- Amazon EFS
- Google Cloud Filestore
- Azure NetApp Files



