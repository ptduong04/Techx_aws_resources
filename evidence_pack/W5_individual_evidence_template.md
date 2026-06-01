# W5 Individual Evidence Pack

## 1. Submission Overview

| Field | Value |
|---|---|
| Project | W5 Individual Resubmission |
| AWS account type | Personal AWS account |
| AWS Region | `us-west-2` - United States (Oregon) | 
| Student name | Phạm Tùng Dương / XB-DN26-105 |

This evidence pack documents an individual implementation of the W5 AWS project using a cost-conscious, two-VPC architecture. The implementation focuses on the required learning outcomes: private networking, security controls, VPC Flow Logs, storage, backup and restore, API Gateway with Lambda, API key protection, and throttling validation.

The project was built in a personal AWS account and intentionally kept lightweight to reduce cost. Expensive managed components such as NAT Gateway and Network Firewall were not used. Equivalent evidence was collected using route tables, VPC Peering, Security Groups, Network ACLs, VPC Flow Logs, AWS Backup, and API Gateway Usage Plan throttling.

---

## 2. Architecture Summary

### Main Components

| Area | Implementation |
|---|---|
| Networking | Two VPCs connected with VPC Peering |
| App VPC | `xbrain-w5-app-vpc`, CIDR `10.10.0.0/16` |
| Data VPC | `xbrain-w5-data-vpc`, CIDR `10.20.0.0/16` |
| Security | Security Groups, NACL review, no public SSH exposure |
| Observability | VPC Flow Logs delivered to CloudWatch Logs |
| Storage | EFS, DynamoDB, EC2-attached EBS |
| Backup | AWS Backup vault, backup plan, completed jobs, EBS restore verification |
| API | API Gateway REST API integrated with Lambda proxy |
| Access control | API key required through Usage Plan |
| Throttling | API Gateway Usage Plan throttling validated with parallel requests |

---

## 3. Cost Guardrail

Before deploying the technical resources, a monthly AWS Budget was configured as a cost safety control for the personal AWS account.

| Setting | Value |
|---|---|
| Budget name | `xbrain-w5-solo-budget` |
| Budget type | Monthly cost budget |
| Budget amount | `$20` |
| Scope | All AWS services |
| Alert recipient | Configured in AWS Budget |
| Alert thresholds | Actual cost and forecast-based alerts |

The budget is used as a guardrail only. It does not stop resources automatically, so cleanup is still required after evidence collection.

![Budget Detail](../Assets/budget-detail.jpeg)

![Budget Alerts](../Assets/budget-alerts.jpeg)

---

## 4. MH1 - Multi-VPC Connectivity and Flow Logs

### Objective

Build two isolated VPCs, connect them using VPC Peering, configure private routing between CIDR ranges, and enable VPC Flow Logs for network audit evidence.

### Implemented Resources

| Resource | Value |
|---|---|
| App VPC | `xbrain-w5-app-vpc` |
| App VPC CIDR | `10.10.0.0/16` |
| App private subnet A | `10.10.1.0/24` in `us-west-2a` |
| App private subnet B | `10.10.2.0/24` in `us-west-2b` |
| Data VPC | `xbrain-w5-data-vpc` |
| Data VPC CIDR | `10.20.0.0/16` |
| Data private subnet A | `10.20.1.0/24` in `us-west-2a` |
| Data private subnet B | `10.20.2.0/24` in `us-west-2b` |
| Connectivity | VPC Peering between App VPC and Data VPC |
| App route | `10.20.0.0/16` to VPC Peering target |
| Data route | `10.10.0.0/16` to VPC Peering target |
| App Flow Log group | `/aws/vpc/flow-logs/xbrain-w5-app` |
| Data Flow Log group | `/aws/vpc/flow-logs/xbrain-w5-data` |

### Evidence

![App VPC](../Assets/mh1-app-vpc.jpeg)

![Data VPC](../Assets/mh1-data-vpc.jpeg)

![Subnets](../Assets/mh1-subnets.jpeg)

![VPC Peering Active](../Assets/mh1-peering-active.jpeg)

![App Route Table](../Assets/mh1-app-route-table.jpeg)

![Data Route Table](../Assets/mh1-data-route-table.jpeg)

![App Flow Logs](../Assets/mh1-app-flow-logs.jpeg)

![Data Flow Logs](../Assets/mh1-data-flow-logs.jpeg)

![Flow Logs Sample](../Assets/mh1-flow-log-sample.jpeg)

### Result

The App VPC and Data VPC were successfully created as separate private network domains. VPC Peering was activated and both sides were routed explicitly using the peer VPC CIDR blocks. VPC Flow Logs were enabled for both VPCs and CloudWatch Logs confirmed that traffic records were being captured.

---

## 5. MH2 - Security Group and NACL Hardening

### Objective

Demonstrate least-privilege network security using Security Groups, NACL review, and negative traffic validation through VPC Flow Logs.

### Implemented Security Controls

| Control | Implementation |
|---|---|
| Lambda Security Group | No inbound rules required |
| EFS Security Group | Allows NFS TCP `2049` only from the Lambda security group |
| EC2 Test Security Group | No public SSH from `0.0.0.0/0` |
| NACL Review | App and Data subnet NACLs reviewed as subnet-level controls |
| Negative test | EC2 traffic to EFS NFS port was blocked |
| Reject evidence | VPC Flow Logs showed `REJECT OK` records |

The EC2 test instance was intentionally placed in a private subnet with no public IPv4 address. This validates that administration and tests were performed without exposing SSH to the internet.

### Evidence

![Lambda Security Group](../Assets/mh2-lambda-sg.jpeg)

![EFS Security Group](../Assets/mh2-efs-sg.jpeg)

![EC2 Security Group](../Assets/mh2-ec2-sg.jpeg)

![App NACL](../Assets/mh2-app-nacl.jpeg)

![Data NACL](../Assets/mh2-data-nacl.jpeg)

![Negative Test](../Assets/mh2-negative-test.jpeg)

![Flow Logs Reject](../Assets/mh2-flow-log-reject.jpeg)

### Result

The network security posture follows least privilege. Lambda has no inbound exposure, EFS accepts NFS only from the intended application security group, and EC2 has no public SSH access. The negative traffic test produced `REJECT` records in VPC Flow Logs, confirming that blocked traffic is observable and auditable.

---

## 6. MH3 - File Storage, Backup, and Restore

### Objective

Create storage resources, protect them with AWS Backup, and prove that a backup can be restored and verified from an EC2 instance.

### Implemented Resources

| Resource | Value |
|---|---|
| DynamoDB table | `xbrain-w5-items` |
| DynamoDB partition key | `id` |
| EFS file system | `xbrain-w5-efs` |
| EFS access pattern | Lambda mounted EFS through access point at `/mnt/efs` |
| EC2 test instance | `xbrain-w5-ec2-test` |
| EBS test volume | `1 GiB` attached to EC2 |
| Test file | `/mnt/w5-ebs/test.txt` |
| Backup vault | `xbrain-w5-backup-vault` |
| Backup plan | `xbrain-w5-backup-plan` |
| Backup resources | DynamoDB, EFS, and EBS |
| Restore test | Restored EBS recovery point to a new EBS volume |

### DynamoDB Evidence

The DynamoDB table was created in on-demand mode and populated with a test item to prove basic data persistence.

![DynamoDB Table](../Assets/mh3-dynamodb-table.jpeg)

![DynamoDB Item](../Assets/mh3-dynamodb-item.jpeg)

### EFS Evidence

The EFS file system was created in the App VPC with mount targets in two Availability Zones. Lambda was later configured to mount the EFS access point and write/read a test file.

![EFS Detail](../Assets/mh3-efs-detail.jpeg)

![EFS Mount Targets](../Assets/mh3-efs-mount-targets.jpeg)

![EFS Read Write](../Assets/mh3-efs-read-write.jpeg)

### EC2 and EBS Evidence

A private EC2 instance was used to prepare a small EBS test volume. The volume was formatted, mounted, and populated with a test file before backup.

![EC2 Test](../Assets/mh3-ec2-test.jpeg)

![EBS Test File](../Assets/mh3-ebs-test-file.jpeg)

### AWS Backup and Restore Evidence

AWS Backup was configured to protect the storage resources. On-demand backup jobs completed successfully for DynamoDB, EFS, and EBS. The EBS recovery point was restored to a new volume, attached back to the EC2 test instance, mounted without reformatting, and verified by reading the original `test.txt` file.

![Backup Plan](../Assets/mh3-backup-plan.jpeg)

![Backup Jobs Completed](../Assets/mh3-backup-jobs-completed.jpeg)

![Restore Job Completed](../Assets/mh3-restore-job-completed.jpeg)

![Restored EBS Verify](../Assets/mh3-restored-ebs-verify.jpeg)

### Result

The storage and backup workflow was validated end to end. The completed backup jobs prove AWS Backup coverage across DynamoDB, EFS, and EBS. The restore test proves that the backed-up EBS data was recoverable and readable from a restored volume.

---

## 7. MH4 - API Gateway in Front of Lambda

### Objective

Expose the Lambda function through API Gateway, protect the API with an API key and usage plan, and validate both authorized and unauthorized request behavior.

### Implemented Resources

| Resource | Value |
|---|---|
| Lambda function | `xbrain-w5-api-handler` |
| Runtime | Python 3.12 |
| EFS mount path | `/mnt/efs` |
| API Gateway type | REST API |
| API name | `xbrain-w5-api` |
| Resource path | `/items` |
| Method | `GET` |
| Integration | Lambda proxy integration |
| Stage | `dev` |
| Invoke URL | `https://n0mp4nxflg.execute-api.us-west-2.amazonaws.com/dev/items` |
| API key | `xbrain-w5-api-key` |
| Usage plan | `xbrain-w5-usage-plan` |

The API key value is intentionally not documented in this evidence pack.

### Evidence

![Lambda Detail](../Assets/mh4-lambda-detail.jpeg)

![API Gateway Routes](../Assets/mh4-api-routes.jpeg)

![API Stage](../Assets/mh4-api-stage.jpeg)

![API Key](../Assets/mh4-api-key.jpeg)

![Usage Plan](../Assets/mh4-usage-plan.jpeg)

![Curl Without API Key - 403](../Assets/mh4-curl-403.jpg)

![Curl With API Key - 200](../Assets/mh4-curl-200.jpg)

### Result

API Gateway was successfully configured as the public entry point for the Lambda function. The `/items` method requires an API key. A request without the `x-api-key` header returned `403`, while a request with a valid API key returned `200` and invoked the Lambda function successfully. The Lambda response also confirmed EFS write/read behavior through the mounted file system.

---

## 8. MH5 - Serverless Scaling and Throttling Control

### Objective

Demonstrate a serverless traffic control pattern and capture throttling evidence under parallel request load.

### Implementation Note

The original plan was to use Lambda reserved concurrency. However, this personal AWS account did not allow setting reserved concurrency because the account-level unreserved concurrency could not go below AWS required limits. To keep the validation aligned with the W5 objective, API Gateway Usage Plan throttling was used instead.

This is a valid serverless throttling control because API Gateway can protect the downstream Lambda function by limiting request rate and burst capacity before requests reach Lambda.

### Implemented Throttling Control

| Control | Value |
|---|---|
| Throttling layer | API Gateway Usage Plan |
| Usage plan | `xbrain-w5-usage-plan` |
| Test configuration | Rate `1 request/second`, Burst `1 request` |
| Test method | Parallel requests from local PowerShell/Postman |
| Expected result | Some requests succeed, excess requests return `429` |
| Monitoring evidence | API Gateway `4XXError` metric increased in CloudWatch |

### Evidence

![Throttle Control Configuration](../Assets/mh5-reserved-concurrency.jpeg)

![Parallel Test](../Assets/mh5-parallel-test.jpg)

![CloudWatch Throttles](../Assets/mh5-cloudwatch-throttles.jpeg)

![Lambda Logs](../Assets/mh5-lambda-log.jpeg)

### Result

The parallel request test generated throttled API responses. CloudWatch showed an increase in API Gateway `4XXError` metrics, which corresponds to rejected/throttled requests such as HTTP `429`. Lambda logs also confirmed that successful requests continued to invoke the function normally. This demonstrates that the serverless endpoint can be protected from uncontrolled request bursts.



