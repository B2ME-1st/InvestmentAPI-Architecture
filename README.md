# Investment Order Microservice – Reliability & Deployment Design

## Overview
This document outlines the deployment architecture and reliability design for a critical microservice responsible for processing investment orders in a high-availability fintech environment.

The design focuses on scalability, reliability, fault tolerance, and operational visibility to ensure stable transaction processing during both normal and peak trading periods.

### Assumptions
For the purpose of this design, the following assumptions are made:

1. The service will initially support only Nigerian residents.
2. The platform is expected to experience periodic spikes in investment requests, especially during stock market opening hours on weekdays.
3. NDPR regulations permit the hosting of investment-related data on AWS cloud infrastructure.

---

## Request Flow Diagram
> See the image below for the request flow diagram.

 <img width="1082" height="489" alt="FLOW" src="https://github.com/user-attachments/assets/c974399d-91b1-4a57-8d92-569ffdb1f188" />


### Request Lifecycle

1. Users: Clients or frontend applications initiate the trade by submitting an investment order payload via a secure HTTPS request.

2. API GW (API Gateway): Acts as the entry gate to authenticate the request, enforce rate limits, and safely route the traffic into the private network.

3. Lambda Ingress: A highly scalable, serverless function that performs rapid validation, hands off the order id checking, and offloads the request instantly.

4. Redis (Idempotency): Intercepts the request inside Lambda to verify the unique idempotency key, immediately blocking duplicate submissions to prevent double-ordering. Successful order ids rreturn a 200 to the user.

5. Apache Kafka (Multi-AZ): Acts as an immutable, highly available shock absorber that durably saves the "Order Placed" event across multiple zones without risk of data loss.

6. ECS Fargate Workers: A steady-state container fleet that pulls messages from Kafka at a regulated pace to safely execute core investment and clearing business logic.

7. Aurora PostgreSQL: The final, ACID-compliant relational database where the transaction is securely and permanently recorded with strict data integrity.

---

## System Design (Architecture)
To guarantee zero data loss and continuous availability for a high-volume stock exchange model in Nigeria, this architecture decouples synchronous API ingestion from asynchronous order execution using a message broker.

> See image below for the full architecture diagram.

<img width="1637" height="977" alt="archi drawio" src="https://github.com/user-attachments/assets/5a9dd5d6-01db-4a27-b06e-8cc60511bb55" />

---

## Design Justification

### Compute Design

- **AWS Lambda (Ingress Layer):**
    - *Justification:* Selected for its ability to scale instantly and incur zero cost when idle (e.g., during market closures). Lambda handles basic payload validation, checks for duplicate requests using Redis, submits events to Kafka, and returns an HTTP 202 Accepted response to clients.

- **AWS ECS Fargate (Worker Microservice):**
    - *Justification:* Ideal for running the core execution logic, as it supports persistent database connections and efficient connection pooling—something Lambda cannot do well. Fargate consumes messages from Kafka at a controlled rate, preventing resource spikes and enabling a smaller, optimized compute footprint.
 
 - **Primary Region: Europe Ireland**
    - *Justification:* This region was chosen due to it's low latency to Nigeria for various applications and use cases.

---

### Database Design

- **Amazon Aurora PostgreSQL (Multi-AZ Cluster):**
    - *Justification:* Required for strict ACID compliance and relational integrity, which are essential for financial transactions. Aurora PostgreSQL is preferred over standard RDS due to its separation of compute and storage, enabling independent scaling and sub-100ms replication.

- **Amazon ElastiCache for Redis (Shared Cache):**
    - *Justification:* Used at the ingress layer to store client `idempotency_keys` with a 2-hour TTL. This ensures duplicate client requests are blocked at the edge, before reaching Kafka or the database, thereby safeguarding data integrity.


---

## Failover Strategy

> See the image below for the failover architecture diagram.
<img width="1892" height="1328" alt="FAILOVER-ARCH drawio" src="https://github.com/user-attachments/assets/bc19058c-56e1-40b7-8908-ee26e4a45c7b" />


_The **API Gateway** plays a critical role in the scenario for a need to failover to a replica site_


### Multi-AZ Deployment (ACTIVE-ACTIVE)
All critical infrastructure components—including ECS Fargate compute, Apache Kafka brokers, and Aurora PostgreSQL—are deployed across three AWS Availability Zones (AZ-A, AZ-B, AZ-C) within the primary region. This ensures that the failure of a single data center does not impact overall service availability.

### Automatic Failover Approach
- **Compute/Ingress:** Inherently Multi-AZ. AWS handles the routing out of the box.
- **ECS Fargate:** You configure your ECS Service to distribute tasks evenly across 3 private subnets (AZ-A, AZ-B, AZ-C). Fargate automatically scales up or down across these zones. If AZ-A fails, the Application Load Balancer / API Gateway shifts traffic entirely to AZ-B and AZ-C instantly.
- **Database (Aurora):** You run an Aurora cluster with 1 Writer instance in AZ-A and 2 Read Replicas (one in AZ-B, one in AZ-C).
Failover: If AZ-A dies, Aurora automatically promotes one of the read replicas to become the primary Writer in under 30 seconds. Your application layer handles this seamlessly using the Aurora cluster endpoint.
- **Queue (Kafka):** Kafka’s replication factor of 3 ensures that if a broker fails, a new partition leader is elected from a healthy AZ, maintaining message durability and availability.
- **Redis: ** Deploy AWS ElastiCache Redis in Cluster Mode across 3 AZs with automatic failover enabled.

### Recovery Expectations
- **Application Ingress:** Instantaneous traffic rerouting (0 seconds).
- **Database Failover:** Replica promotion and DNS propagation occur in under 30 seconds.
- **Data Loss:** Zero data loss during local failover, as Kafka requires confirmations from at least two AZs (`min.insync.replicas=2`) before acknowledging order ingestion.

---

## Disaster Recovery (Across Different Regions)
### Secondary Region (ACTIVE-PASSIVE) 
> See the image below for the DR strategy

<img width="1832" height="1337" alt="dr-recovery drawio" src="https://github.com/user-attachments/assets/58ebc395-3441-4d51-b560-36bca4a06598" />

We will use an **Active-Passive** (Warm Standby) strategy in a Europe London region, for the rare cases where the entire primary AWS Region goes offline

To survive a total AWS regional outage, an Aurora Global Database asynchronously replicates data to a secondary DR region (RPO < 1 second). Infrastructure is defined via Terraform, but are scaled down to save costs and ready to scale up via a Route 53 DNS switch within 15 minutes (RTO < 15 mins).

---

## Backup & Recovery Strategy
### Backup Frequency
- **Continuous Backups:** Aurora is configured for automated, continuous backups with incremental transaction log recording.
- **Daily Snapshots:** Full storage snapshots are taken every 24 hours during off-peak hours.

### Snapshot Strategy
- **Multi-layered Snapshots:** Daily automated snapshots are copied to an isolated, immutable AWS Backup vault in a separate backup-specific AWS account. This protects against accidental or malicious deletion in the primary account.

### Retention Policy
- **Continuous/PITR Data:** Retained for 35 days, enabling point-in-time recovery.
- **Long-term Snapshots:** Retained for 7 years to meet financial compliance and regulatory requirements.
- **Kafka Logs:** Message logs are retained for 7 days, allowing replay in case of downstream failures.

### RPO/RTO Targets
- **Recovery Point Objective (RPO):**
    - Standard ingestion failover: 0 seconds (no data loss).
    - Database point-in-time restore: <5 minutes (restore to the exact second before data corruption).
- **Recovery Time Objective (RTO):**
    - Standard ingestion failover: <30 seconds (automatic recovery from AZ loss).
    - Database point-in-time restore: <2 hours (restore from backup and application repointing).

---


## Service Level Objectives

For investments, **data correctness and consistency** (ACID compliance) trump raw speed. An investor is far more concerned that their ₦20 million investment order is processed accurately and consistently than whether the request completed in 300ms instead of 900ms.

> Else, you go explain tire (in court).

That being said, these are the 2 choices for my SLOs for this service and why;

### 1. Order Durability (99.999%)

- **Goal:** Never lose a queued message.
- **Implementation:** 
    - Use high-availability storage such as multi-AZ Kafka replication and synchronous database commits.
- **Rationale:** 
    - Investors care more about the certainty of their ₦3M order being processed than about minor latency differences.
- **Measurement:** 
    - Metrics: `orders_ingested / orders_persisted_to_kafka`
    - Monitoring: Grafana dashboard comparing generated order IDs to successfully queued IDs.
    - **Alert:** Any drop below 99.999% triggers an alert.

### 2. Ingestion Availability (99.95%)

- **Goal:** Maximize API uptime.
- **Implementation:** 
    - Monitor HTTP metrics from AWS API Gateway or Lambda ingestion endpoints using Prometheus.
- **Rationale:** 
    - Slow responses are inconvenient, but downtime during market events is unacceptable.
- **Measurement:** 
    - Metrics: `(sum of 2xx and 3xx HTTP requests) / (total HTTP requests)`
    - **Alert:** Any significant drop below 99.95% triggers an alert.

---

**Summary:**  
_We guarantee our doors are always open to accept your trade (**Availability**), and once you walk through the door, your trade cannot be lost (**Durability**)._


---

## Reliability Considerations & Trade-offs

1. **Consistency vs. Latency:**  
     - Prioritize data correctness over ultra-fast responses. Synchronous replication adds minor delays but prevents order loss or duplication.

2. **Operational Complexity vs. Resilience:**  
     - Multi-layered system (Lambda, Redis, Kafka, Fargate) increases complexity but ensures resilience and continuous order acceptance.

3. **Idle Cloud Cost vs. High Availability:**  
     - Higher AWS costs due to multi-AZ deployments are justified by the need for continuous uptime and automatic failover.


---

## Conclusion
This design aims to provide a reliable, scalable, and observable foundation for handling critical investment transactions while maintaining strong operational resilience during peak usage periods.

*I hope you enjoyed reading through this document as much as I enjoyed designing and thinking through the reliability considerations behind it.*
