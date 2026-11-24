# Campus Notes Board

## 1. Overview and Architecture Diagram

Campus Notes Board is a web application designed to help students upload, view, and download course-related PDF notes. It uses Amazon EC2 to host the application, Amazon RDS for MySQL to store note information and metadata, and Amazon S3 to store uploaded PDF documents.

The application runs on an Ubuntu EC2 instance, with Nginx acting as a reverse proxy. The EC2 instance uses an IAM role to access S3 securely without requiring permanent AWS access keys on the server.

### Architecture Diagram

![Campus Notes Board Architecture](screenshots/architecture.png)

### AWS Resources

| Resource | Name / Configuration |
|---|---|
| AWS Region | Europe (Stockholm) – `eu-north-1` |
| VPC | `campus-notes-vpc` |
| Internet Gateway | `campus-notes-igw` |
| EC2 Instance | `campus-notes-app` |
| EC2 Security Group | `app-sg` |
| RDS Instance | `campus-notes-db` |
| RDS Security Group | `db-sg` |
| IAM Role | `notes-ec2-role` |
| Database | `campus_notes` |
| Database User | `notes_app` |
| S3 Bucket | Your configured project bucket |

### Architecture Flow

1. Students access the application through a web browser.
2. Nginx receives incoming HTTP requests and forwards them to the application.
3. The application communicates with Amazon RDS to store and retrieve note metadata.
4. PDF files are uploaded to and downloaded from Amazon S3.
5. The EC2 instance uses its attached IAM role to access S3.
6. Amazon CloudWatch monitors selected infrastructure metrics.
7. AWS CloudTrail records account activity and management events.

## 2. Prerequisites

The following are required to deploy and operate the project:

### AWS Requirements

- An AWS account or authorized AWS Skill Builder project environment.
- Access to the AWS Management Console.
- Permissions to create and manage VPC, EC2, RDS, S3, IAM, CloudWatch, and CloudTrail resources.
- AWS Region: `eu-north-1` (Europe – Stockholm).

### Software and Tools

- Ubuntu Linux EC2 instance.
- Git and GitHub account.
- MySQL client for database connectivity testing.
- Nginx web server.
- Application runtime and dependencies required by the starter project.
- AWS Systems Manager Session Manager or another authorized method of connecting to the EC2 instance.
- A PDF document for testing uploads and downloads.

### Repository

GitHub Repository:

https://github.com/AWS-SBG-Cloud-and-Devops-Track-AASTU/campus-notes-board

## 3. Deployment Steps

The deployment follows the sequence below, starting with network infrastructure and ending with a running web application.

### Step 1: Create the VPC Infrastructure

Create the VPC named `campus-notes-vpc`.

The network infrastructure includes:

- One public subnet for the application EC2 instance.
- Two private subnets intended for private resources and the RDS subnet group.
- An Internet Gateway named `campus-notes-igw`.
- A public route table with an internet route through the Internet Gateway.
- A private route table for private subnet communication.

Configure the route table associations and security groups according to the network design.

Refer to [`docs/network.md`](docs/network.md) for the network configuration and subnet details.

### Step 2: Create Security Groups

Create the following security groups:

**Application Security Group (`app-sg`)**

Controls inbound and outbound traffic for the EC2 instance.

Configure the required web access and the administration method used to connect to the instance.

**Database Security Group (`db-sg`)**

Controls access to the RDS database.

The intended inbound rule allows MySQL traffic from the application security group:

| Type | Protocol | Port | Source |
|---|---|---|---|
| MySQL/Aurora | TCP | 3306 | `app-sg` |

The database should not be directly accessible from the public internet.

### Step 3: Launch the EC2 Instance

Launch an Ubuntu EC2 instance with the following configuration:

- Instance name: `campus-notes-app`
- Operating system: Ubuntu Linux
- VPC: `campus-notes-vpc`
- Subnet: Public subnet
- Security group: `app-sg`
- IAM instance profile: `notes-ec2-role`

Ensure that the instance is running and that the selected administration method works.

### Step 4: Prepare the EC2 Environment

Connect to the EC2 instance through AWS Systems Manager Session Manager.

Update the system packages:

```bash
sudo apt update
sudo apt upgrade -y
```

Install the required software:

```bash
sudo apt install -y git nginx mysql-client
```

Install the additional runtime and application dependencies required by the starter project.

### Step 5: Clone the Application Repository

Clone the project repository:

```bash
git clone https://github.com/AWS-SBG-Cloud-and-Devops-Track-AASTU/campus-notes-board.git
```

Navigate into the project directory:

```bash
cd campus-notes-board
```

Install the application dependencies using the package manager and instructions provided by the project.

### Step 6: Configure Environment Variables

Configure the application environment variables required for the database and S3 connection.

Store sensitive configuration in an environment file outside the Git repository.

Example location:

```text
/etc/notes/notes.env
```

Never commit passwords, secret keys, or other sensitive configuration values to GitHub.

### Step 7: Configure the Application Service

Create a dedicated Linux service user for the application.

Configure a systemd service named `notes` using the correct application startup command and working directory.

Reload systemd and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable notes
sudo systemctl start notes
```

Check the service status:

```bash
sudo systemctl status notes --no-pager
```

Check the application logs:

```bash
sudo journalctl -u notes -n 100 --no-pager
```

### Step 8: Configure Nginx

Configure Nginx as a reverse proxy to forward incoming HTTP requests to the application running locally on the EC2 instance.

Test the Nginx configuration:

```bash
sudo nginx -t
```

Restart and enable Nginx:

```bash
sudo systemctl restart nginx
sudo systemctl enable nginx
```

### Step 9: Verify the Deployment

Check the application's health endpoint:

```bash
curl http://localhost/health
```

The endpoint should report the application and its required service connections, including the database and S3, according to the application's implementation.

Open the EC2 instance's public IP address in a browser to access the Campus Notes Board application.

## 4. Configuration Variables

The application uses environment variables to configure its database and S3 connections.

| Variable | Meaning |
|---|---|
| `DB_HOST` | RDS database endpoint |
| `DB_PORT` | MySQL database port |
| `DB_NAME` | Database name |
| `DB_USER` | Application database username |
| `DB_PASSWORD` | Password for the application database user |
| `S3_BUCKET` | S3 bucket used to store PDF documents |
| `AWS_REGION` | AWS Region of the S3 bucket |
| `MAX_UPLOAD_MB` | Maximum accepted PDF upload size (optional) |

### Environment File Security

The environment file should be stored outside the project repository.

Example:

```text
/etc/notes/notes.env
```

The file should be owned by the appropriate administrative account and readable only by the application service account and root.

Example permissions, assuming the service group is named `notes`:

```bash
sudo chown root:notes /etc/notes/notes.env
sudo chmod 640 /etc/notes/notes.env
```

The actual owner and group should match the Linux user and group configured for the application.

Sensitive values must never be committed to GitHub.

## 5. Database and S3 Setup

### Amazon RDS – MySQL

Amazon RDS for MySQL is used to store the metadata associated with uploaded notes.

Database configuration:

| Setting | Value |
|---|---|
| DB Instance Identifier | `campus-notes-db` |
| Database Engine | MySQL |
| Port | `3306` |
| Database Name | `campus_notes` |
| Application User | `notes_app` |
| Region | `eu-north-1` |

The application connects to RDS through the configured database endpoint.

A separate application database user, `notes_app`, is used instead of relying on the database master account.

The application database user should have only the privileges required by the application.

### Notes Table

The `notes` table stores the metadata associated with uploaded PDF documents.

The expected fields include:

| Column | Purpose |
|---|---|
| `id` | Unique identifier for each note |
| `title` | Title of the uploaded note |
| `course_name` | Related course name |
| `description` | Description of the note |
| `s3_object_key` | Location key of the PDF object in S3 |
| `created_at` | Date and time the note was created |

The actual table structure should match the application's database schema.

### Amazon S3

Amazon S3 is used to store the PDF documents uploaded through Campus Notes Board.

The bucket is configured to keep the uploaded documents private.

The intended security settings include:

- Block Public Access enabled.
- Bucket ownership configured to prevent public or external ownership issues.
- Server-side encryption enabled using SSE-S3.
- No public bucket policy or public object permissions.
- Uploaded documents organized under the `notes/` prefix.

The database stores the S3 object key associated with each note, while the actual PDF file is stored in the S3 bucket.

The application accesses S3 through the IAM role attached to the EC2 instance.

## 6. Security

Security is implemented through network restrictions, IAM permissions, private database access, and secure storage configuration.

### Security Groups

**`app-sg` – Application Security Group**

Controls traffic to and from the EC2 application instance.

It allows the required application traffic and the configured administration method.

**`db-sg` – Database Security Group**

Restricts access to the RDS database.

The intended inbound MySQL rule is:

| Type | Protocol | Port | Source |
|---|---|---|---|
| MySQL/Aurora | TCP | 3306 | `app-sg` |

This ensures that the database accepts connections from the application security group rather than unrestricted public sources.

### IAM Instance Role

The EC2 instance uses the IAM role:

`notes-ec2-role`

The role provides temporary AWS credentials to the EC2 instance through its instance profile.

This allows the application to communicate with AWS services without storing permanent access keys on the server.

The role includes the required permissions for:

- AWS Systems Manager access, through `AmazonSSMManagedInstanceCore`, if Session Manager is used.
- Application access to the required S3 bucket and objects.

### S3 IAM Policy

The application-specific IAM policy should grant only the permissions required by the application.

| IAM Action | Resource | Purpose |
|---|---|---|
| `s3:ListBucket` | S3 bucket ARN | Allows listing bucket objects when required |
| `s3:PutObject` | `bucket/notes/*` | Allows uploading PDF documents |
| `s3:GetObject` | `bucket/notes/*` | Allows downloading PDF documents |

The policy should use the actual S3 bucket ARN.

This follows the principle of least privilege by restricting access to the required bucket and object prefix.

### Additional Security Measures

- RDS is not intended to be publicly accessible.
- S3 Block Public Access is enabled.
- Database credentials are stored outside the Git repository.
- The EC2 instance uses an IAM role rather than hardcoded AWS access keys.
- Security groups restrict communication between the application and database.
- CloudTrail provides an audit history of supported AWS management events.

## 7. Evidence

Screenshots documenting the project implementation, AWS infrastructure configuration, deployment process, security settings, monitoring, and testing are available in the repository's `screenshots/` directory.

The screenshots provide visual evidence of the following:

- VPC, subnet, and route table configurations.
- EC2 instance configuration and deployment.
- Amazon RDS database configuration.
- Amazon S3 bucket settings and uploaded PDF objects.
- IAM roles, policies, and permissions.
- Application health checks and end-to-end functionality.
- PDF upload and download operations.
- S3 private-access verification.
- CloudWatch monitoring and alarm configuration.
- CloudTrail management events.
- Troubleshooting exercises and recovery.
- AWS Pricing Calculator cost estimate.

All supporting screenshots can be found in the [`screenshots/`](screenshots/) directory.


## 8. Known Limitations, Cost and Cleanup

### Known Limitations

Campus Notes Board is designed as a student project and learning application.

Its limitations depend on the features implemented in the starter application.

Potential limitations to document where applicable include:

- The application is designed for a limited project environment rather than large-scale production traffic.
- The application supports PDF documents rather than arbitrary file types.
- The application depends on the availability of its EC2 instance, RDS database, and S3 bucket.
- The deployment uses HTTP unless HTTPS has been separately configured.
- Authentication, authorization, and file access restrictions depend on the features actually implemented in the application.

### AWS Cost Estimation

The estimated infrastructure cost was calculated using the **AWS Pricing Calculator** for the Europe (Stockholm) Region (`eu-north-1`).

The estimate assumes the following configuration:

| AWS Service | Configuration | Monthly Estimate |
|---|---|---:|
| Amazon EC2 | 1 Linux `t3.micro` instance, On-Demand, 730 hours/month | $7.88 |
| Amazon EBS | 1 gp3 volume, 30 GB, including configured snapshot assumptions | $8.20 |
| Amazon RDS for MySQL | 1 `db.t4g.micro` instance, 20 GB gp3 storage, Single-AZ, 730 hours/month | $14.08 |
| Amazon S3 | 1 GB Standard storage, 1,000 write/list requests, 5,000 read/other requests | $0.03 |
| Amazon CloudWatch | Configured monitoring assumptions and 1 standard-resolution alarm metric | $0.10 |
| Amazon VPC | 1 in-use public IPv4 address | $3.65 |
| **Total** | | **$33.94** |

**Estimated Costs Summary**

| Cost Type | Amount |
|---|---:|
| Upfront Cost | $0.00 |
| Monthly Cost | $33.94 |
| Total 12-Month Cost | $407.28 |

The EBS estimate includes the snapshot configuration specified in the calculator: two snapshots per day and 3 GB changed per snapshot.

The project environment provided a $100 AWS credit allowance. This credit allocation is separate from the estimated infrastructure cost.

The AWS Pricing Calculator provides an estimate rather than an actual billing statement. Actual charges may vary depending on resource usage, storage consumption, snapshots, data transfer, requests, taxes, and how long the resources remain active.

The estimated monthly cost assumes that the EC2 and RDS instances run for 730 hours per month. Actual charges may be lower if resources are stopped or deleted when they are not needed. Storage and other applicable charges may still apply.

### Cleanup

After completing the project assessment and saving all required documentation and evidence, unused AWS resources should be removed to avoid unnecessary charges.

The cleanup process includes:

1. Stop or terminate the EC2 instance when it is no longer required.
2. Back up any required database information and delete the RDS instance when it is safe to do so.
3. Empty the S3 bucket before deleting it.
4. Remove unused security groups, subnets, route tables, and the VPC after checking their dependencies.
5. Detach and delete unused IAM policies and roles.
6. Remove unused CloudWatch alarms and log groups.
7. Verify that no unnecessary billable AWS resources remain active.

**Important:** Do not delete project resources before the assessment or demonstration if they are still required. Save all screenshots and documentation before performing cleanup.

---

## Project Repository

GitHub: https://github.com/AWS-SBG-Cloud-and-Devops-Track-AASTU/campus-notes-board
