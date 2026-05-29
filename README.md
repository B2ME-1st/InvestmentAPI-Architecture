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


### Infrastructure Components
[Placeholder]

Describe:
- Compute layer
- Networking
- Load balancing
- Database layer
- Cache layer
- Observability stack
- Backup systems
- CI/CD integration

---

## Design Justification

### Compute Design
[Placeholder]

Explain:
- Choice of compute platform
- Scaling approach
- High availability considerations
- Container orchestration decisions

---

### Database Design
[Placeholder]

Explain:
- Database choice
- Replication strategy
- Read/write considerations
- Connection management
- Data durability considerations

---

### Failover Strategy
[Placeholder]

Explain:
- Multi-AZ deployment
- Automatic failover approach
- Recovery expectations
- Service redundancy
- Disaster recovery considerations

---

### Backup & Recovery Strategy
[Placeholder]

Explain:
- Backup frequency
- Snapshot strategy
- Retention policy
- RPO/RTO targets
- Restore testing approach

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
