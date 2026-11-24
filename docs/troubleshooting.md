# Troubleshooting Documentation

## Campus Notes Board — AWS Gate 2

This document describes three intentional infrastructure failures introduced during the deployment and testing of the Campus Notes Board application. Each exercise demonstrates the ability to identify, investigate, troubleshoot, and resolve common AWS infrastructure problems.

The three scenarios cover:

1. RDS MySQL connectivity and security group configuration.
2. S3 object access and IAM permissions.
3. VPC routing and Internet Gateway connectivity.

---

## 1. RDS Database Connectivity Failure

### 1.1 Original Configuration

The Campus Notes Board application uses Amazon RDS for MySQL to store note metadata. The database is deployed in private subnets within the project VPC and is not publicly accessible.

The EC2 application instance communicates with the database through private networking.

The original security group configuration was:

| Configuration | Value |
|---|---|
| Database | campus-notes-db |
| Database engine | MySQL |
| Port | 3306 |
| Database security group | db-sg |
| Application security group | app-sg |
| Inbound rule | MySQL/Aurora |
| Source | app-sg |
| Public accessibility | Disabled |

### 1.2 Failure Simulation

To simulate a database connectivity failure, the inbound MySQL rule allowing traffic from `app-sg` was temporarily removed from `db-sg`.

This prevented the EC2 instance from establishing a connection to the RDS database.

### 1.3 Observed Symptoms

When attempting to connect from the EC2 terminal, the following command was executed:

```bash
mysql -h campus-notes-db.cbekmmka6gdu.eu-north-1.rds.amazonaws.com \
-P 3306 -u dbadmin -p
```

The connection returned:

```text
ERROR 2003 (HY000):
Can't connect to MySQL server
(110)
```

The application health endpoint also reported a database connectivity failure.

### 1.4 Investigation

The following potential causes were investigated:

- RDS database availability and status.
- Correctness of the database endpoint and port.
- Security group inbound rules.
- EC2 and RDS VPC configuration.
- Network connectivity between the application and database subnets.

The investigation identified the missing MySQL inbound security group rule as the cause of the failure.

### 1.5 Resolution

The missing inbound rule was restored in the `db-sg` security group.

The corrected configuration was:

| Field | Configuration |
|---|---|
| Type | MySQL/Aurora |
| Protocol | TCP |
| Port | 3306 |
| Source | app-sg |

The source was configured using the application security group reference instead of allowing unrestricted access from public IP addresses.

### 1.6 Verification

The database connection was tested again from the EC2 instance:

```bash
MYSQL_HISTFILE=/dev/null mysql \
-h campus-notes-db.cbekmmka6gdu.eu-north-1.rds.amazonaws.com \
-P 3306 \
-u notes_app \
-p campus_notes
```

The connection was considered successful when the MySQL monitor opened.

The application health endpoint was also tested:

```bash
curl http://localhost/health
```

Expected database status:

```text
database: ok
```

### 1.7 Lesson Learned

Security groups control network access between AWS resources. Even when EC2 and RDS are deployed in the same VPC, the required inbound and outbound permissions must be configured correctly.

Using security group references instead of unrestricted IP access provides a more secure and maintainable configuration.

---

## 2. S3 Object Access Failure

### 2.1 Original Configuration

The application uses Amazon S3 to store uploaded PDF documents. The EC2 instance accesses the bucket through an attached IAM role rather than manually configured AWS access keys.

The original IAM configuration included:

| IAM Action | Purpose |
|---|---|
| `s3:ListBucket` | Allows bucket-level access checks and listing |
| `s3:PutObject` | Allows PDF uploads |
| `s3:GetObject` | Allows PDF retrieval |

The object permissions were restricted to the application's `notes/` prefix.

The S3 bucket also had Block Public Access enabled.

### 2.2 Failure Simulation

To simulate an S3 access failure, the `s3:GetObject` permission was temporarily removed from the custom IAM policy attached to the EC2 instance role.

The application could still access the bucket and perform operations permitted by its remaining permissions, but it could no longer retrieve uploaded PDF objects.

### 2.3 Observed Symptoms

When attempting to download an existing PDF through the application, the request failed.

The application logs were inspected using:

```bash
sudo journalctl -u notes -n 80 --no-pager
```

The expected error was an AWS S3 `AccessDenied` response associated with the missing `GetObject` permission.

Bucket-level health checks could still succeed because `s3:ListBucket` had not been removed.

### 2.4 Investigation

The following areas were examined:

- IAM role attached to the EC2 instance.
- Permissions attached to the IAM role.
- S3 bucket name and configured Region.
- Object ARN and permitted object prefix.
- S3 Block Public Access settings.

The investigation identified the missing `s3:GetObject` action as the cause of the failed download.

### 2.5 Resolution

The missing permission was restored to the custom IAM policy.

The corrected object permission was:

```json
{
  "Sid": "NotesObjectAccess",
  "Effect": "Allow",
  "Action": [
    "s3:PutObject",
    "s3:GetObject"
  ],
  "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/notes/*"
}
```

The bucket name was replaced with the actual project bucket name.

The bucket-level permission `s3:ListBucket` was retained separately.

### 2.6 Verification

The application was tested again by downloading the previously uploaded PDF.

Successful retrieval confirmed that the EC2 instance had regained the required S3 object permission.

The bucket remained private throughout the test.

### 2.7 Lesson Learned

IAM permissions determine which AWS resources an application can access and which operations it can perform.

Removing a single permission can break a specific application feature even when other AWS operations continue working.

Following the principle of least privilege ensures that the application receives only the permissions required for its functionality.

---

## 3. VPC Routing Failure

### 3.1 Original Configuration

The Campus Notes Board infrastructure uses a custom VPC with public and private subnets.

The public subnet contains the EC2 application instance and uses an Internet Gateway for external connectivity.

The original public route table contained:

| Destination | Target |
|---|---|
| VPC CIDR | local |
| 0.0.0.0/0 | Internet Gateway |

The Internet Gateway was attached to the project VPC.

### 3.2 Failure Simulation

To simulate a routing failure, the default internet route was temporarily removed from the public route table.

The removal of the route prevented the public subnet from forwarding internet-bound traffic through the Internet Gateway.

This could interrupt external connectivity to the EC2 instance and its Systems Manager communication.

### 3.3 Observed Symptoms

The expected symptoms included:

- Loss of internet connectivity from the EC2 instance.
- Inability to access the application through its public IPv4 address.
- Possible interruption of the Systems Manager Session Manager connection.
- Failure of external connectivity checks.

### 3.4 Investigation

The following components were examined:

- Public route table.
- Default route destination.
- Internet Gateway attachment.
- Public subnet route table association.
- EC2 public IPv4 address.
- Network connectivity.

The investigation identified the missing default route as the cause of the connectivity problem.

### 3.5 Resolution

The original default route was restored in the public route table.

The corrected route configuration was:

| Destination | Target |
|---|---|
| VPC CIDR | local |
| 0.0.0.0/0 | Original Internet Gateway |

The route was restored through the AWS Management Console.

If the Session Manager connection was interrupted, the console was used to repair the route before reconnecting to the instance.

### 3.6 Verification

After restoring the route, the EC2 instance was checked for connectivity.

The following checks were performed:

```bash
curl http://localhost/health
```

The public application address was also opened in a browser.

Successful access to the application and recovery of the health endpoint confirmed that the routing configuration had been restored.

### 3.7 Lesson Learned

Route tables determine how network traffic moves between subnets and external networks.

A missing default route can prevent a public subnet from reaching the internet even when an Internet Gateway is attached to the VPC.

Infrastructure changes should always be performed carefully, with the original configuration documented before modification.

---

## 4. Summary of Troubleshooting Exercises

| Failure | Root Cause | Resolution |
|---|---|---|
| RDS connectivity | Missing MySQL inbound rule | Restored TCP 3306 access from app-sg |
| S3 object access | Missing `s3:GetObject` permission | Restored the required IAM action |
| VPC routing | Missing default internet route | Restored `0.0.0.0/0` to the Internet Gateway |

## 5. Overall Lessons Learned

The troubleshooting exercises demonstrated the importance of correctly configuring AWS networking, access control, and infrastructure components.

The main lessons include:

1. **Network security:** Security groups must explicitly allow the communication required between application and database resources.

2. **Least privilege:** IAM roles should contain only the permissions necessary for application functionality.

3. **Routing:** Correct route table configuration is essential for communication between public subnets and external networks.

4. **Monitoring and investigation:** Application logs, health endpoints, AWS Console configurations, and connectivity tests help identify infrastructure failures.

5. **Controlled recovery:** Infrastructure failures should be introduced individually, documented, investigated, and reversed after testing.

## 6. Conclusion

The three troubleshooting exercises provided practical experience in diagnosing common cloud infrastructure problems.

By deliberately modifying security group rules, IAM permissions, and VPC routing configurations, the project demonstrated how individual AWS components affect application availability.

The successful restoration of the original configurations reinforced the importance of secure infrastructure design, least-privilege access, proper network configuration, and systematic troubleshooting.

---

**Project:** Campus Notes Board  
**Platform:** Amazon Web Services (AWS)  
**Region:** Europe (Stockholm) — eu-north-1  
**Environment:** EC2, RDS MySQL, S3, IAM, VPC, CloudWatch and CloudTrail  
