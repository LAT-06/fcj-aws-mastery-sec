
# Introduction & Context

### AWS Well-Architected Security Pillar
- Why Encryption Matters
-  Encryption - _the last line of defense_
# AWS KMS FUNDAMENTALS
- encryption trong AWS: **AWS Key Management Service (KMS)**
1. **KMS là gì:** dịch vụ quản lý khóa tập trung cho tất cả dịch vụ AWS.
2. **Key types:** AWS Managed Key, Customer Managed Key (CMK).
3. **Envelope Encryption:** Data Key (mã hóa dữ liệu) được mã hóa bằng CMK.
4. **Key Rotation & Access Control:** tự động xoay key và kiểm soát qua IAM + Key Policy.
- “How Envelope Encryption Works” (sơ đồ Data → Data Key → CMK → Ciphertext).
# Encryption At Rest
### S3 Encryption

- **SSE-S3:** AWS quản lý key tự động.
    
- **SSE-KMS:** bạn kiểm soát key qua KMS.
    
- **SSE-C:** bạn tự upload key mỗi lần (ít dùng).

### EBS Encryption
- “Mỗi EBS volume có thể được mã hóa khi tạo — bao gồm cả snapshot, backup, logs.”
    
- “KMS key được gán riêng cho volume hoặc sử dụng default key.”
    
- “Bật _encryption by default_ ở cấp account là best practice.”
    
- Mọi dữ liệu (bao gồm swap và metadata) đều được mã hóa.


### EFS Encryption
- “EFS là dịch vụ lưu trữ file chia sẻ, tương tự như network drive.”
    
- Có **hai lớp bảo vệ:**
    
    1. **At rest:** mã hóa bằng KMS key.
        
    2. **In transit:** mã hóa bằng TLS khi EC2 mount EFS.
        
- “Bật cả hai lớp này sẽ đảm bảo toàn bộ dữ liệu trên EFS luôn được bảo vệ.”
    
- “EFS còn hỗ trợ IAM + NFS permission + Security Group port 2049 để kiểm soát truy cập.”

### Database Encryption – RDS & DynamoDB
- **RDS:**
    
    - Encryption at rest phải bật khi tạo DB instance.
        
    - Hỗ trợ logs, backups, snapshots encryption.
        
    - Encryption in transit bằng SSL.
        
- **DynamoDB:**
    
    - Encryption at rest tự động bật.
        
    - Có thể chọn CMK riêng.
        
- “Luôn bật SSL connections và lưu credentials trong Secrets Manager.”

# Secrete Management
### AWS Secrets Manager

- Lưu và xoay password tự động (ví dụ RDS).
    
- Tích hợp IAM và CloudTrail để audit truy cập.
### AWS Certificate Manager (ACM)
- Quản lý SSL/TLS certs tự động

# Demo
- **Tạo KMS CMK:**
    
    - KMS → Customer Managed Key → đặt alias `project-encryption-key`.
        
- **S3:**
    
    - Tạo bucket `secure-demo-bucket`, bật SSE-KMS bằng key vừa tạo.
        
    - Upload file test và xem metadata encryption.
        
- **EBS:**
    
    - Tạo EC2 instance với volume đã bật encryption.
        
    - Kiểm tra encryption trong console.
        
- **EFS:**
    
    - Tạo file system → enable encryption at rest (KMS key) → mount vào EC2 qua TLS (`-o tls`).
        
    - Lưu file test vào `/mnt/efs`.
        
- **RDS:**
    
    - Tạo DB instance bật encryption bằng KMS key.
        
    - Truy cập qua SSL connection.
        
- **Secrets Manager:**
    
    - Lưu DB credentials → đọc lại qua AWS CLI.
