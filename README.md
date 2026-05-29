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

---

### Database Design

- **Amazon Aurora PostgreSQL (Multi-AZ Cluster):**
    - *Justification:* Required for strict ACID compliance and relational integrity, which are essential for financial transactions. Aurora PostgreSQL is preferred over standard RDS due to its separation of compute and storage, enabling independent scaling and sub-100ms replication.

- **Amazon ElastiCache for Redis (Shared Cache):**
    - *Justification:* Used at the ingress layer to store client `idempotency_keys` with a 2-hour TTL. This ensures duplicate client requests are blocked at the edge, before reaching Kafka or the database, thereby safeguarding data integrity.


---

## Failover Strategy

### Multi-AZ Deployment
All critical infrastructure components—including ECS Fargate compute, Apache Kafka brokers, and Aurora PostgreSQL—are deployed across three AWS Availability Zones (AZ-A, AZ-B, AZ-C) within the primary region. This ensures that the failure of a single data center does not impact overall service availability.

### Automatic Failover Approach
- **Compute/Ingress:** AWS API Gateway automatically reroutes traffic away from any failed AZ, ensuring requests are only sent to healthy Lambda and ECS Fargate instances.
- **Database (Aurora):** Aurora automatically promotes a multi-AZ read replica to primary if the current writer fails, minimizing downtime.
- **Queue (Kafka):** Kafka’s replication factor of 3 ensures that if a broker fails, a new partition leader is elected from a healthy AZ, maintaining message durability and availability.

### Recovery Expectations
- **Application Ingress:** Instantaneous traffic rerouting (0 seconds).
- **Database Failover:** Replica promotion and DNS propagation occur in under 30 seconds.
- **Data Loss:** Zero data loss during local failover, as Kafka requires confirmations from at least two AZs (`min.insync.replicas=2`) before acknowledging order ingestion.

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

## Service Level Objectives (SLOs)

### Availability SLO
[Placeholder]

---

### Latency SLO
[Placeholder]

---

### Error Rate SLO
[Placeholder]

---

### Data Integrity SLO
[Placeholder]

---

## Observability & Alerting

### Metrics & Monitoring
[Placeholder]

Describe:
- Prometheus metrics collection
- RED/USE metrics
- Infrastructure metrics
- Database metrics

---

### Dashboards
[Placeholder]

Describe:
- Grafana dashboards
- Business metrics
- Infrastructure health views
- Incident visibility dashboards

---

### Alerting Strategy
[Placeholder]

Describe:
- Prometheus alert rules
- Alert severity levels
- Escalation flow
- On-call considerations

---

## Reliability Considerations
[Placeholder]

Describe:
- Single points of failure eliminated
- Scaling bottlenecks
- Traffic surge handling
- Operational risks
- Future improvements

---

## Conclusion
[Placeholder]

Summarize the reliability, scalability, and operational goals of the architecture.
