# AWS KMS
## Overview
- KMS là service bảo mật lõi (core security service) 
	- tạo, quản lý và kiểm soát khóa mã hóa (sử dụng trên toàn hệ thống aws)
- Được xây dựa trên model "centralized key store + hardware security module (HSM)" nghĩa là tất cả các khóa của bạn được bảo vệ bởi phần cứng đạt chuẩn **FIPS 140-2 Level 3** 
 KMS key hay là encryption key nó là 1 chuỗi bit (binary string) hoặc base64
và nó nằm trong các phần cứng dùng trong HSM và các HSM nằm trong các datacenter vật lý của aws 

## KMS Architecture
1. **Components trong KMS:**

- **CMK (Customer Master Key)** hoặc **KMS Key**: khóa chính được tạo trong KMS, là “root” để mã hóa các khóa khác.
	- Trong thực tế thì CMK được tạo và lưu trong KMS (trong HSM)
	- Mỗi CMK có:
		- Key ID (UUID duy nhất)
		- Key policy
		- Alias
		- Metadata
	- Chức năng:
		- - **Generate Data Key** (tạo ra khóa con tạm thời để mã hóa dữ liệu).
		- **Encrypt / Decrypt Data Key** (để bảo vệ Data Key).
---
- **Data Key:** khóa tạm thời sinh ra từ CMK để mã hóa dữ liệu thực tế.
    - khi yêu cầu
```bash
   aws kms generate-data-key \
  --key-id alias/myapp-key \
  --key-spec AES_256
  ```
    - AWS KMS trả về:
		- **Plaintext data key** – dùng để mã hóa dữ liệu ngay trong ứng dụng.
		- **Encrypted data key** – là bản đã được CMK mã hóa, bạn lưu lại để sau này giải mã.
---
- **Key Policy:** chính sách xác định ai được phép làm gì với key.
```json	
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAdmins",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
      "Action": "kms:*",
      "Resource": "*"    
      }
  ]
}
```
- Version
	- Gần như luôn là "2012-10-17" — định dạng JSON policy của AWS.
- ID
	- Tùy chọn. Chỉ là **định danh nội bộ** cho policy, giúp bạn phân biệt nếu có nhiều version.
- Statement
	- Là **danh sách các điều khoản (rule)** mà bạn định nghĩa.
	- Mỗi phần tử trong `Statement` là một **object** gồm các thành phần sau:

| Thuộc tính | Ý nghĩa                                                                                                  |
| ---------- | -------------------------------------------------------------------------------------------------------- |
| Sid        | (Statement ID) – tên hoặc mô tả ngắn gọn, giúp dễ đọc.                                                   |
| Effect     | `Allow` hoặc `Deny` – quyết định cho phép hay từ chối.                                                   |
| Principal  | Ai được áp dụng quyền này (user, role, service, account, root, v.v.)                                     |
| Action     | Những hành động được phép làm với key (VD: `kms:Encrypt`, `kms:Decrypt`, `kms:CreateGrant`...)           |
| Resource   | Mục tiêu áp dụng (thường là `"*"` vì policy chỉ áp dụng cho chính CMK này).                              |
| Condition  | (Tuỳ chọn) – điều kiện ràng buộc thêm, ví dụ: chỉ cho phép khi request từ VPC cụ thể hoặc region cụ thể. |

---
- **Grants:** quyền tạm thời cho ứng dụng hoặc dịch vụ khác sử dụng key mà không cần sửa policy.
	- - Policy là “chính sách gốc” – thay đổi cần cẩn thận, có thể ảnh hưởng toàn hệ thống.
	- Grants giúp tạo quyền “vay mượn tạm thời” trong các workflow tự động (VD: S3, EBS, Lambda).
	- Ex: Khi S3 muốn mã hóa dữ liệu bằng key của bạn, KMS tạo 1 **grant** tạm cho S3 dùng key đó.
```bash
aws kms create-grant \
  --key-id alias/myapp-key \
  --grantee-principal arn:aws:iam::account-id:role/service-role/s3-role \
  --operations Encrypt Decrypt
```
- Grant sẽ tồn tại đến khi bị revoke hoặc key bị xóa.
---
- **Alias:** tên định danh dễ nhớ của key (VD: `alias/myapp-key`).
	- thay vì _1234abcd-56ef-78gh-90ij-klmnopqrstuv_ thì chỉ cần gọi _alias/myapp-key_

**2. Key Lifecycle:**

- Tạo key → dùng key → audit và rotation → có thể disable hoặc xóa key.

## KMS Key Types - phân biệt rõ
| Type                           | Description                                            | Managed By | Use Case                                                                       |
| ------------------------------ | ------------------------------------------------------ | ---------- | ------------------------------------------------------------------------------ |
| **AWS Managed Key**            | AWS tự tạo, dùng cho dịch vụ mặc định                  | AWS        | Dùng cho encryption đơn giản (vd bật encryption trên S3 mà không chỉ định key) |
| **Customer Managed Key (CMK)** | Người dùng tạo, quản lý toàn bộ policy, rotation, xóa  | You        | Khi bạn cần kiểm soát chi tiết (vd dữ liệu khách hàng, dự án thật)             |
| **AWS Owned Key**              | AWS sở hữu & quản lý hoàn toàn (bạn không thấy key ID) | AWS        | Dùng cho dữ liệu tạm hoặc metadata nội bộ AWS                                  |
## Envelope Encryption
### Tổng quát các bước

- Có 1 file *data.txt* cần được mã hóa
#### 1. Gọi KMS và yêu cầu tạo Data Key
```bash
aws kms generate-data-key \
  --key-id alias/myproject-key \
  --key-spec AES_256 \
  --region ap-southeast-1
```
KMS trả về:
- **Plaintext data key** (dạng base64)
- **Encrypted data key** (được mã hóa bằng CMK)
#### 2. Mã hóa 
Dùng **Plaintext data key** để mã hóa dữ liệu thật (`data.txt` → `data.enc`)
#### 3. Xóa Plaintext
Ứng dụng xóa Plaintext key khỏi bộ nhớ và chỉ lưu lại **Encrypted data key** cạnh dữ liệu.
#### 4. Giải mã
Khi cần giải mã → ứng dụng gửi Encrypted data key cho KMS → KMS giải mã → trả lại Plaintext data key → giải mã file.

### Tại sao dùng cơ chế này?
- **Hiệu năng cao:** vì mã hóa dữ liệu không cần gọi API KMS nhiều lần.
    
- **An toàn:** CMK không bao giờ rời khỏi HSM.
    
- **Linh hoạt:** có thể lưu trữ Encrypted Data Key ở bất kỳ đâu.

## Key Rotation (only for CMK)
- Thay đổi định kỳ khóa mã hóa (CMK) => giảm rủi ro bị lộ hoặc là bị cooked bởi mấy cu hackẻ 
#### Automation Key Rotation
- Bạn bật **automatic rotation** cho CMK trong console hoặc CLI.
    
- AWS sẽ **tạo một phiên bản key mới mỗi 1 năm** (12 tháng).
    
- Dữ liệu cũ vẫn **có thể giải mã** được, vì CMK lưu trữ thông tin metadata về các phiên bản trước đó.

#### Manual Key Rotation
- Tạo một CMK mới.
    
- Dùng key mới để **mã hóa lại toàn bộ dữ liệu cũ** (re-encrypt).
    
- Xóa key cũ khi không cần thiết nữa.

|Tiêu chí|Automatic Rotation|Manual Rotation|
|---|---|---|
|Áp dụng|Customer Managed Key|Customer Managed Key|
|Tần suất|1 năm / lần|Tùy bạn|
|Can thiệp của người dùng|Không cần|Phải thao tác thủ công|
|Giải mã dữ liệu cũ|Tự động|Bạn phải re-encrypt để dùng key mới|
|Rủi ro|Thấp, ít sai sót|Cao nếu thao tác sai|
- **Automatic rotation = tiện lợi, an toàn**, đủ dùng cho hầu hết use case.
    
- **Manual rotation = kiểm soát cực chặt**, dùng khi có yêu cầu compliance hoặc regulatory.
    
- **Best practice:** bật automatic rotation cho tất cả CMK quan trọng.

## Auditing & Monitoring
KMS **không mã hóa ngẫu nhiên**, mà mỗi hoạt động đều được log:

- **AWS CloudTrail:** ghi lại mọi thao tác với key (ai gọi, lúc nào, từ đâu).
    
- **AWS Config:** kiểm tra compliance (ví dụ bucket nào chưa bật SSE-KMS).
    
- **CloudWatch Metrics:** số lần gọi API, tỷ lệ thất bại, v.v.
    

> Trong môi trường doanh nghiệp, CloudTrail + Config là combo bắt buộc khi triển khai KMS.

---
## Lab
Flow lab:
### **Chuẩn bị**

- Tạo **CMK** trên Console (`alias/test-encryption-key`)
    
- Tạo file test `data.txt` với nội dung plaintext.
    
- Cấu hình **AWS CLI** với IAM user có quyền Key Admin + Key User.
    

---

### **Generate Data Key từ CMK**

- **AWS KMS sinh ra 2 thứ**:
    
    1. **Plaintext Data Key** → dùng để encrypt file.
        
    2. **CiphertextBlob (Encrypted Data Key)** → Data Key được mã hóa bằng CMK, lưu lại để giải mã sau.
        

> Đây là **Envelope Encryption**: CMK không trực tiếp mã hóa file, chỉ mã hóa Data Key.

---

### **Encrypt file**

- Dùng **Plaintext Data Key** với `openssl` (hoặc SDK) để encrypt file plaintext → file ciphertext (`data.enc`)
    
- Lưu `CiphertextBlob` (`datakey.encrypted`) để sau này decrypt.
    

---

### **Decrypt Data Key**

- Khi muốn decrypt file:
    
    1. Gửi `CiphertextBlob` (`datakey.encrypted`) lên KMS → KMS trả về **Plaintext Data Key**.
        
    2. Lưu Data Key này tạm thời vào file `datakey.bin`.
        

---

### **Decrypt file**

- Dùng Data Key vừa nhận để giải mã file ciphertext → thu được file plaintext gốc (`data_decrypted.txt`).
    

---

### **Cleanup**

- Xóa các file sensitive (`datakey.bin`)
    
- Giữ ciphertext file (`data.enc`) để lưu trữ hoặc trao đổi an toàn.
 ---
 
- Choose KMS
![[Pasted image 20251111154205.png]]

- Choose CMK & Create key
![[Pasted image 20251111154344.png]]

- Choose Symmetric
![[Pasted image 20251111154914.png]]
- Symmetric Key (Khóa đối xứng)
	- **Chỉ có một khóa duy nhất** dùng cho cả **mã hóa (encrypt)** và **giải mã (decrypt)**.
	- Key này phải được **giữ bí mật tuyệt đối**.
```mathematica
Plaintext --[Symmetric Key]--> Ciphertext
Ciphertext --[Same Symmetric Key]--> Plaintext
```
- Asymmetric Key (Khóa bất đối xứng)
	- **Có 2 khóa khác nhau nhưng liên kết với nhau**:
		1. **Public Key** – để mã hóa dữ liệu, có thể chia sẻ cho bất kỳ ai.
		2. **Private Key** – để giải mã dữ liệu, phải giữ bí mật tuyệt đối.
```mathematica
Plaintext --[Public Key]--> Ciphertext
Ciphertext --[Private Key]--> Plaintext
```

- Choose a key alias
![[Pasted image 20251111160059.png]]

- Choose key administrators key
![[Pasted image 20251111160504.png]]
- **Key Administrators** = những **người hoặc nhóm chịu trách nhiệm quản lý CMK (Customer Master Key)** trong AWS KMS.

- Chuẩn bị aws cli
![[Pasted image 20251111161824.png]]

- Tạo 1 file txt (test thử thôi)
![[Pasted image 20251111162359.png]]

- Generate Data Key và lưu Plaintext Data Key
![[Pasted image 20251111162541.png]]

- Lưu Ciphertext data key (được cmk encrypt)
![[Pasted image 20251111162700.png]]

- Encrypt file sử dụng openssl
![[Pasted image 20251111162758.png]]

