# ☁️ AWS Cloud Computing — Lab Notes & Cheat Sheet

---

## 📋 TABLE OF CONTENTS

| # | Experiment | Week |
|---|-----------|------|
| 1 | VPC with NAT Gateway & Bastion Host | Week 6 |
| 2 | S3 → Lambda → DynamoDB | Week 7 |
| 3 | SNS + SQS + S3 Event | Week 8 |
| 4 | ELB + Auto Scaling | Week 9 |
| 5 | Elastic Beanstalk | Week 10 |
| 6 | Amazon Lex Chatbot | Week 11 |
| 7 | IAM User & CLI | Week 12 |
| 8 | IAM Role for EC2 | Week 13 |

---
---

# 🧪 EXPERIMENTS

---

## EXPERIMENT 1 — VPC with NAT Gateway & Bastion Host (Week 6)

### 🎯 AIM
Create a secure VPC with public and private subnets and enable internet access to private instances using a NAT Gateway and Bastion Host.

### 🛠️ Services Used
`Amazon VPC` · `EC2` · `Internet Gateway` · `NAT Gateway`

### 📋 Prerequisites
- AWS Account
- Region: `ap-south-1`
- Key pairs: `bastion.pem`, `dbserver.pem`

### 🏗️ Architecture / Flow
```
Internet → Bastion (Public Subnet) → Private EC2 → NAT Gateway
```

### 🌐 Network Details

| Resource | Value |
|----------|-------|
| VPC CIDR | `10.0.0.0/16` |
| Public Subnet | `10.0.1.0/24` |
| Private Subnet | `10.0.2.0/24` |
| Internet Gateway | Attached to Public Subnet |
| NAT Gateway | Placed in Public Subnet, routes Private traffic |

---

### 📌 PROCEDURE

#### PART A — Setup Network

**Step 1:** Go to AWS Console

**Step 2:** Navigate to `Services → VPC → Create VPC`

**Step 3:** Configure VPC
```
Name tag  → MyVPC
IPv4 CIDR → 10.0.0.0/16
```
Click **Create VPC**

**Step 4:** Create Subnets (`VPC → Subnets → Create subnet`)

| Subnet | Name | CIDR |
|--------|------|------|
| Public | `PublicSubnet` | `10.0.1.0/24` |
| Private | `PrivateSubnet` | `10.0.2.0/24` |

**Step 5:** Create Internet Gateway (`VPC → Internet Gateways → Create`)
```
Name → MyIGW
Attach to VPC → MyVPC
```

**Step 6:** Configure Public Route Table (`VPC → Route Tables → Create`)
```
Name → PublicRT
Route: Destination → 0.0.0.0/0  |  Target → Internet Gateway (MyIGW)
Associate with → PublicSubnet
```

---

#### PART B — NAT Gateway

**Step 7:** Create NAT Gateway (`VPC → NAT Gateways → Create`)
```
Name    → MyNAT
Subnet  → PublicSubnet
Elastic IP → Allocate New
```
Click **Create**

**Step 8:** Create Private Route Table
```
Name → PrivateRT
Route: Destination → 0.0.0.0/0  |  Target → NAT Gateway (MyNAT)
Associate with → PrivateSubnet
```

---

#### PART C — EC2 Instances

**Step 9:** Launch Bastion Server (`EC2 → Launch Instance`)
```
Name              → Bastion
AMI               → Ubuntu
Subnet            → PublicSubnet
Auto-assign IP    → Enable
Key pair          → bastion.pem
```

**Step 10:** Launch Private Server
```
Name              → DBServer
Subnet            → PrivateSubnet
Auto-assign IP    → Disable (No public IP)
Key pair          → dbserver.pem
```

---

#### PART D — Connectivity

```bash
# Step 1: SSH into Bastion
ssh -i bastion.pem ubuntu@<Bastion_Public_IP>

# Step 2: Copy DB key to Bastion
scp -i bastion.pem dbserver.pem ubuntu@<Bastion_Public_IP>:/home/ubuntu/

# Step 3: From Bastion, SSH into Private EC2
ssh -i dbserver.pem ubuntu@10.0.2.x
```

---

### ✅ Testing / Verification

```bash
# Inside private EC2
ping google.com
# ✔ Internet works via NAT Gateway
```

| Test | Result |
|------|--------|
| SSH into Bastion | ✅ Success |
| SSH into Private EC2 (via Bastion) | ✅ Success |
| Internet access from Private EC2 | ✅ Works via NAT |

### 📌 Result
Private EC2 instance accesses the internet securely via NAT Gateway without a public IP.

### ⚠️ Common Errors

| Error | Fix |
|-------|-----|
| NAT Gateway placed in private subnet | NAT must be in **public** subnet |
| Missing route in PrivateRT | Add `0.0.0.0/0 → NAT Gateway` |
| Security group blocking SSH | Allow port 22 in SG inbound rules |

---
---

## EXPERIMENT 2 — S3 → Lambda → DynamoDB (Week 7)

### 🎯 AIM
Trigger a Lambda function on S3 file upload and store file details in DynamoDB.

### 🛠️ Services Used
`S3` · `Lambda` · `DynamoDB` · `IAM`

### 🏗️ Flow
```
S3 (Upload) → Lambda (Trigger) → DynamoDB (Store Record)
```

---

### 📌 PROCEDURE

#### PART A — Create S3 Bucket
`Services → S3 → Create bucket`
```
Name       → my-lambda-bucket-123
Region     → ap-south-1
Versioning → Enable
```

#### PART B — Create DynamoDB Table
`DynamoDB → Create table`
```
Table name     → Newtable
Partition key  → id (String)
```

#### PART C — Create Lambda Function
`Lambda → Create function`
```
Name     → S3TriggerFunction
Runtime  → Python 3.x
Role     → Create new role with basic Lambda permissions
```

#### PART D — Add S3 Trigger
```
Add trigger → S3
Bucket      → my-lambda-bucket-123
Event type  → All object create events
```

#### PART E — Lambda Code (Python)

```python
import boto3
from uuid import uuid4

def lambda_handler(event, context):
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Newtable')

    for record in event['Records']:
        table.put_item(
            Item={
                'id': str(uuid4()),
                'bucket': record['s3']['bucket']['name'],
                'object': record['s3']['object']['key'],
                'event': record['eventName']
            }
        )

    return {"statusCode": 200}
```

#### PART F — IAM Policy for Lambda → DynamoDB

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:PutItem",
      "Resource": "arn:aws:dynamodb:ap-south-1:<ACCOUNT-ID>:table/Newtable"
    }
  ]
}
```

---

### ✅ Testing

| Step | Action | Expected |
|------|--------|----------|
| 1 | Upload any file to S3 bucket | Lambda triggers |
| 2 | Check DynamoDB → Newtable | New record with id, bucket, object |

### 📌 Result
Lambda successfully stores S3 event metadata in DynamoDB upon every file upload.

---
---

## EXPERIMENT 3 — SNS + SQS + S3 Event (Week 8)

### 🎯 AIM
Implement S3 event-driven messaging using SNS and SQS.

### 🛠️ Services Used
`S3` · `SNS` · `SQS` · `Lambda`

### 🏗️ Flow
```
S3 (Upload) → SNS (Publish) → SQS (Queue) → Lambda (Consume)
```

---

### 📌 PROCEDURE

#### PART A — Create SNS Topic
`SNS → Create topic`
```
Name → MyS3SNSTopic
Type → Standard
```

#### PART B — Create SQS Queue
`SQS → Create queue`
```
Name → MyS3Queue
Type → Standard
```

#### PART C — Subscribe SQS to SNS
`SNS → Subscriptions → Create subscription`
```
Protocol → SQS
Endpoint → ARN of MyS3Queue
```

#### PART D — SQS Access Policy (Allow SNS to send messages)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSNS",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:ap-south-1:<ACCOUNT-ID>:MyS3Queue",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:sns:ap-south-1:<ACCOUNT-ID>:MySNSTopic"
        }
      }
    }
  ]
}
```

#### PART E — SNS Access Policy (Allow S3 to publish)

```json
{
  "Version": "2012-10-17",
  "Id": "AllowS3ToPublish",
  "Statement": [
    {
      "Sid": "AllowS3Publish",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:ap-south-1:<ACCOUNT-ID>:MySNSTopic",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::mys3eventbucket123"
        },
        "StringEquals": {
          "aws:SourceAccount": "<ACCOUNT-ID>"
        }
      }
    }
  ]
}
```

#### PART F — Configure S3 Event Notification
`S3 Bucket → Properties → Event notifications → Create`
```
Event type  → All object create events
Destination → SNS Topic → MyS3SNSTopic
```

#### PART G — Lambda Consumer for SQS (Python)

```python
def lambda_handler(event, context):
    for record in event['Records']:
        print(record['body'])

    return {"statusCode": 200}
```

---

### ✅ Testing

| Step | Action | Expected |
|------|--------|----------|
| 1 | Upload file to S3 | S3 publishes to SNS |
| 2 | SNS → SQS | Message appears in queue |
| 3 | SQS → Lambda | Lambda logs the message |

### 📌 Result
Event-driven architecture S3 → SNS → SQS → Lambda works successfully.

---
---

## EXPERIMENT 4 — ELB + Auto Scaling (Week 9)

### 🎯 AIM
Distribute traffic across EC2 instances using an Application Load Balancer (ALB) and Auto Scaling Group (ASG).

### 🏗️ Flow
```
User → ALB (Port 80) → Target Group → EC2 Instances (ASG)
```

### ⚙️ Key Values

| Setting | Value |
|---------|-------|
| Port | `80` |
| Health check path | `/` |
| ASG Min | `1` |
| ASG Desired | `2` |
| ASG Max | `4` |

---

### 📌 PROCEDURE

#### PART A — EC2 Web Server Setup (User Data / Manual)

```bash
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "This is Server 1" > /var/www/html/index.html
```

#### PART B — Create Application Load Balancer
`EC2 → Load Balancers → Create → Application Load Balancer`
```
Listener  → HTTP : 80
Subnets   → Select at least 2 availability zones
```

#### PART C — Create Target Group
```
Type      → Instance
Protocol  → HTTP : 80
Health check path → /
Register instances → Add your EC2s
```

#### PART D — Create Auto Scaling Group
```
Launch Template → Use AMI of configured EC2
Min Capacity    → 1
Desired         → 2
Max Capacity    → 4
Attach to       → Target Group (ALB)
```

---

### ✅ Testing
1. Copy ALB DNS Name → Open in browser
2. Refresh multiple times → Different server responses confirm load balancing

### 📌 Result
Traffic is dynamically distributed across EC2 instances. ASG scales based on demand.

---
---

## EXPERIMENT 5 — Elastic Beanstalk (Week 10)

### 🎯 AIM
Deploy a web application using Elastic Beanstalk without managing infrastructure.

---

### 📌 PROCEDURE

1. Go to `Elastic Beanstalk → Create Application`
2. Create Environment → `Web server environment`
3. Platform → **Java** (or Python/Node as needed)
4. Upload `.war` (or `.zip`) application file
5. Click **Create Environment** — AWS provisions EC2, ALB, etc. automatically

---

### 📌 Result
Application is deployed and accessible via the auto-generated Elastic Beanstalk URL.

---
---

## EXPERIMENT 6 — Amazon Lex Chatbot (Week 11)

### 🎯 AIM
Create a conversational chatbot using Amazon Lex.

### 🤖 Bot Configuration

| Setting | Value |
|---------|-------|
| Bot Name | `HotelBookingBot` |
| Intent | `BookHotel` |
| Slot: Age | Guest age |
| Slot: Location | City/location |
| Slot: Nights | Number of nights |

---

### 📌 PROCEDURE

1. Go to `Amazon Lex → Create bot`
2. Name → `HotelBookingBot`
3. Create Intent → `BookHotel`
4. Add Slots:
   - `age` (Number)
   - `location` (String)
   - `nights` (Number)
5. Add sample utterances (e.g., *"I want to book a hotel"*)
6. Build & Test bot

---

### 📌 Result
Chatbot correctly collects hotel booking details through conversation.

---
---

## EXPERIMENT 7 — IAM User & CLI (Week 12)

### 🎯 AIM
Create an IAM user, attach S3 policy, and access AWS via CLI.

---

### 📌 PROCEDURE

1. Go to `IAM → Users → Create user`
2. Attach policy → `AmazonS3FullAccess`
3. Create **Access Key** → Download `.csv`
4. Configure CLI:

```bash
aws configure
# Enter: Access Key ID, Secret Access Key, Region (ap-south-1), Output format (json)
```

5. Test CLI access:

```bash
aws s3 ls                          # List all buckets
aws s3 mb s3://my-unique-bucket-name   # Create bucket
aws ec2 describe-instances         # List EC2 instances
aws iam list-users                 # List IAM users
```

---

### 📌 Result
IAM user successfully accesses AWS services via CLI using access keys.

---
---

## EXPERIMENT 8 — IAM Role for EC2 (Week 13)

### 🎯 AIM
Attach an IAM Role to an EC2 instance to access S3 without hardcoded credentials.

---

### 📌 PROCEDURE

1. Go to `IAM → Roles → Create role`
2. Trusted entity → **EC2**
3. Attach policy → `AmazonS3FullAccess`
4. Name role → `EC2S3Role`
5. Go to EC2 instance → `Actions → Security → Modify IAM Role`
6. Attach `EC2S3Role`
7. SSH into EC2 and test:

```bash
aws s3 ls    # Works without aws configure (role provides credentials)
```

---

### 📌 Result
EC2 accesses S3 securely via IAM Role — no access keys needed.

---
---

# 🧠 QUICK REFERENCE CHEAT SHEET

> ⚡ Most commonly asked in viva/exams

---

## 🧠 1. AWS CLI — Most Asked Commands

```bash
aws configure                           # Setup credentials
aws s3 ls                               # List all S3 buckets
aws s3 mb s3://my-unique-bucket-name    # Create new S3 bucket
aws ec2 describe-instances              # List EC2 instances
aws iam list-users                      # List IAM users
```

---

## 🧠 2. EC2 Web Server Setup

```bash
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "This is Server 1" > /var/www/html/index.html
```

---

## 🧠 3. SSH Connection Commands

```bash
# Connect to public EC2
ssh -i key.pem ec2-user@<public-ip>

# Copy private key to Bastion
scp -i bastion.pem dbserver.pem ubuntu@<bastion-ip>:/home/ubuntu/

# SSH into private EC2 from Bastion
ssh -i dbserver.pem ubuntu@10.0.2.x
```

---

## 🧠 4. Lambda Code — S3 → DynamoDB (Python)

```python
import boto3
from uuid import uuid4

def lambda_handler(event, context):
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Newtable')

    for record in event['Records']:
        table.put_item(
            Item={
                'id': str(uuid4()),
                'bucket': record['s3']['bucket']['name'],
                'object': record['s3']['object']['key'],
                'event': record['eventName']
            }
        )

    return {"statusCode": 200}
```

---

## 🧠 5. Lambda Consumer — SQS (Python)

```python
def lambda_handler(event, context):
    for record in event['Records']:
        print(record['body'])

    return {"statusCode": 200}
```

---

## 🔐 6. SNS Access Policy — S3 → SNS (JSON)

```json
{
  "Version": "2012-10-17",
  "Id": "AllowS3ToPublish",
  "Statement": [
    {
      "Sid": "AllowS3Publish",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:ap-south-1:<ACCOUNT-ID>:MySNSTopic",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::mys3eventbucket123"
        },
        "StringEquals": {
          "aws:SourceAccount": "<ACCOUNT-ID>"
        }
      }
    }
  ]
}
```

---

## 🔐 7. SQS Access Policy — SNS → SQS (JSON)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSNS",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:ap-south-1:<ACCOUNT-ID>:MyS3Queue",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:sns:ap-south-1:<ACCOUNT-ID>:MySNSTopic"
        }
      }
    }
  ]
}
```

---

## 🔐 8. Lambda → DynamoDB IAM Policy (JSON)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:PutItem",
      "Resource": "arn:aws:dynamodb:ap-south-1:<ACCOUNT-ID>:table/Newtable"
    }
  ]
}
```

---

## 🔐 9. Basic IAM Policy — S3 Full Access (JSON)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
```

---

## 🌐 10. Network Cheat Sheet — VPC

```
VPC CIDR:        10.0.0.0/16
Public Subnet:   10.0.1.0/24  →  Route: 0.0.0.0/0 → Internet Gateway
Private Subnet:  10.0.2.0/24  →  Route: 0.0.0.0/0 → NAT Gateway
```

| Component | Placement | Purpose |
|-----------|-----------|---------|
| Internet Gateway | VPC | Public internet access |
| NAT Gateway | Public Subnet | Outbound internet for private instances |
| Bastion Host | Public Subnet | SSH jump server to private EC2 |
| DB / App Server | Private Subnet | No direct public exposure |

---

## 🔥 11. Load Balancer + ASG Key Values

```
Protocol:       HTTP
Port:           80
Health Check:   /
ASG Min:        1
ASG Desired:    2
ASG Max:        4
```

---

## 🤖 12. Amazon Lex Quick Setup

```
Bot Name:  HotelBookingBot
Intent:    BookHotel
Slots:     age, location, nights
```

---

## 📊 Services Summary Table

| Service | Purpose | Key Concept |
|---------|---------|-------------|
| VPC | Virtual network | CIDR, Subnets, Route Tables |
| EC2 | Virtual servers | AMI, Key Pair, Security Groups |
| S3 | Object storage | Buckets, Events, Versioning |
| Lambda | Serverless compute | Event-driven, Python handler |
| DynamoDB | NoSQL database | Partition key, PutItem |
| SNS | Pub/Sub messaging | Topic, Subscription |
| SQS | Message queue | Queue, Consumer |
| IAM | Access control | Users, Roles, Policies |
| ALB | Load balancing | Target Group, Listener |
| ASG | Auto scaling | Min/Desired/Max |
| Lex | Chatbot | Intent, Slots, Utterances |
| Beanstalk | PaaS deployment | Upload & deploy |

---

*Notes prepared for AWS Cloud Computing Lab — ap-south-1 (Mumbai)*
