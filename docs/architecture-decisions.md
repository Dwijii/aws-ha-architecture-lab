# Architecture decisions

Each decision lists what I chose, the alternative I rejected, and why.

## 1. Two Availability Zones, not one

- **Chose:** every tier (load balancer, servers, database) spans AZ A and AZ B.
- **Rejected:** a single AZ, which is cheaper and simpler.
- **Why:** an AZ outage would take the whole application down. Tested by failing AZ A in the lab; the app kept running.

## 2. Servers in private subnets behind a load balancer

- **Chose:** the Application Load Balancer sits in public subnets; EC2 servers sit in private subnets.
- **Rejected:** servers with public IP addresses.
- **Why:** only the load balancer is reachable from the internet, which shrinks the attack surface. Servers reach the internet outbound-only through the NAT Gateway.

## 3. Chained security groups

- **Chose:** ALB accepts port 80 from anywhere → app accepts port 80 only from the ALB → database accepts 3306 only from the app.
- **Rejected:** opening ports to `0.0.0.0/0` at every layer.
- **Why:** each layer can only be reached from the one in front of it.
- **Note:** the practice lab simplifies this to a single `web-sg`; the CloudFormation template uses the stricter chained version.

## 4. IAM role, not access keys

- **Chose:** an instance role with `AmazonS3ReadOnlyAccess`.
- **Rejected:** `AmazonS3FullAccess` / `AdministratorAccess`, or access keys stored on the server.
- **Why:** least privilege. Role credentials are temporary and rotate automatically.

## 5. RDS Multi-AZ plus a read replica

- **Chose:** Multi-AZ for failover, plus a separate read replica.
- **Why both:** they solve different problems. Multi-AZ keeps a synchronous standby that you can't read from, for **resilience**. The read replica is an asynchronous copy you *can* read from, for **performance** (reporting queries).
- **Also:** the master password is managed in Secrets Manager (`ManageMasterUserPassword`), and storage is encrypted.

## 6. S3 lifecycle rule

- **Chose:** Standard → Standard-IA after 30 days → Glacier Flexible Retrieval after 90 days.
- **Why:** most files are read heavily when new and rarely afterwards. 30 days is the minimum S3 allows before moving to Standard-IA.

## 7. SNS → SQS fan-out

- **Chose:** one SNS topic (`new-order`) delivering to separate SQS queues per consumer (payments, emails).
- **Rejected:** one shared queue, where each message goes to only one reader, or calling each system directly.
- **Why:** each consumer gets its own copy and processes at its own pace. If one is down, its queue holds messages until it recovers.

## 8. S3 gateway endpoint

- **Chose:** a free gateway VPC endpoint for S3.
- **Why:** server traffic to S3 stays on the AWS network and avoids NAT Gateway data-processing charges.

## Known trade-offs and production next steps

| Area | Current | Production improvement |
| --- | --- | --- |
| NAT Gateway | One, in AZ A (lower cost) | One per AZ, so outbound access survives an AZ failure |
| HTTPS | HTTP on port 80 | ACM certificate and HTTPS listener, redirect 80 → 443 |
| Web protection | None | AWS WAF on the load balancer |
| Scaling | Fixed min 2 / max 4 | Target-tracking policy on CPU or request count |
| Queues | No dead-letter queues | DLQs with alarms for failed messages |
| Monitoring | Default metrics | CloudWatch alarms and dashboards; CloudTrail enabled |
| Content delivery | Served from the load balancer | CloudFront in front for global users |
