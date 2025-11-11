### Cloud Sec Matters
- Move to cloud => share sec responsibility
- AWS gives tools & infra that are *secure by design* but **you still own ur data & configuration**
### The shared responsibility model
| **Responsibility**                         | **AWS** | **Customer (You)** |
| ------------------------------------------ | ------- | ------------------ |
| Physical security (data centers, hardware) | y       | n                  |
| Network and hypervisor security            | y       | n                  |
| OS, applications, and data                 | n       | y                  |
| IAM (users, roles, permissions)            | n       | y                  |
| Encryption keys, access control            | n       | y                  |
=> aws is responsible for the security of the cloud, u are responsible for security in the cloud

Example:
- AWS secures the physical server where your EC2 runs.
- You secure your EC2 instance (patches, firewall, IAM permissions, data encryption).
#### The 5 AWS Well-Architected Pillars

Every good cloud architecture is built around these five pillars:

1. **Operational Excellence** – monitor and improve operations.
    
2. **Security** – protect data, systems, and assets.
    
3. **Reliability** – recover quickly from failures.
    
4. **Performance Efficiency** – use resources efficiently.
    
5. **Cost Optimization** – control spending.

### Layers of Cloud Security
- **Identity & Access Management (IAM)** – who can access what.
    
- **Network Security** – VPCs, firewalls, security groups.
    
- **Data Protection** – encryption, key management, secrets.
    
- **Monitoring & Logging** – detect suspicious activity.
    
- **Compliance & Governance** – meet legal requirements.

### Common Cloud Security Risks
|**Threat**|**Description**|**Example**|
|---|---|---|
|Data Breach|Unauthorized access to data|Public S3 bucket exposing sensitive data|
|Misconfiguration|Human error in setup|Security group open to 0.0.0.0/0|
|Insider Threat|Malicious or careless employee|User exports data without permission|
|Weak Access Control|Poor IAM policy design|Overly permissive `s3:*` on all buckets|
|Lack of Encryption|Storing plain text data|RDS or EBS volume unencrypted|
### Data Classification
Before you encrypt anything, you must **classify your data**.

Typical data classifications:

- **Public:** can be shared freely (e.g., marketing material)
    
- **Internal:** used within the company
    
- **Confidential:** sensitive info like project details
    
- **Restricted:** highly sensitive — financials, PII, healthcare, etc.

### Encryption Overview
There are **two major encryption states** in the cloud:

1. **Encryption at Rest** – protects data stored on disks, S3, EBS, RDS, etc.  
    (Even if someone steals the physical disk, data remains unreadable.)
    
2. **Encryption in Transit** – protects data while it’s moving over the network.  
    (TLS/SSL is used here.)

- **At Rest:** KMS, EBS encryption, RDS encryption, S3 SSE-KMS.
    
- **In Transit:** AWS Certificate Manager (ACM), TLS 1.2/1.3.

### Building a Security-First Mindset
- **Least Privilege:** Give users/services only the permissions they need.
    
- **Defense in Depth:** Don’t rely on one layer — combine IAM + encryption + logging.
    
- **Automation:** Use services like AWS Config and Security Hub to enforce compliance.
    
- **Visibility:** Always know who accessed what (CloudTrail).

AWS doesn’t just give you encryption tools — it gives you the framework to make encryption **repeatable and auditable**.

