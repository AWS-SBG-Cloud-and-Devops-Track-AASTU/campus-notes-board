# Network Plan

## Campus Notes Board — AWS Gate 2

**Region:** Europe (Stockholm) — `eu-north-1`

**VPC Name:** `campus-notes-vpc`

**VPC CIDR:** `10.42.0.0/16`

The Campus Notes Board infrastructure uses a custom Amazon VPC with separate public and private subnets. The public subnet hosts the EC2 application server, while the private subnets provide network placement for the RDS MySQL database.

The network design follows the principle of separating publicly accessible application infrastructure from private database infrastructure.

## Subnets

| Subnet | CIDR | Usable range | Usable IPs | AZ | Purpose | Why this size |
|---|---|---|---:|---|---|---|
| public-subnet | 10.42.1.0/24 | 10.42.1.4 – 10.42.1.254 | 251 | eu-north-1a | Hosts the EC2 application server and nginx. | Provides sufficient addresses for the application server and potential future public resources. |
| private-subnet-a | 10.42.2.0/24 | 10.42.2.4 – 10.42.2.254 | 251 | eu-north-1a | First private subnet for the RDS DB subnet group. | Provides sufficient private address space for database infrastructure. |
| private-subnet-b | 10.42.3.0/24 | 10.42.3.4 – 10.42.3.254 | 251 | eu-north-1b | Second private subnet for the RDS DB subnet group. | Provides a separate Availability Zone for database subnet coverage and future availability improvements. |

### Subnet CIDR Calculations

The VPC uses the CIDR block `10.42.0.0/16`, which provides 65,536 total IPv4 addresses.

Each subnet uses a `/24` CIDR block.

A `/24` subnet contains:

- Total addresses: 256
- AWS-reserved addresses: 5
- Usable addresses: 251

AWS reserves the following five addresses in each subnet:

1. The first address: Network address.
2. The second address: VPC router.
3. The third address: DNS server.
4. The fourth address: Reserved for future use.
5. The last address: Network broadcast address.

For example, the public subnet `10.42.1.0/24` has:

- Network address: `10.42.1.0`
- Reserved addresses: `10.42.1.0 – 10.42.1.3`
- Usable range: `10.42.1.4 – 10.42.1.254`
- Reserved final address: `10.42.1.255`

The same calculation applies to both private subnets.

## Route Tables

| Route table | Routes | Associated subnets |
|---|---|---|
| main | VPC CIDR → local | No explicit subnet associations; left untouched as the main route table. |
| public-rt | VPC CIDR → local; `0.0.0.0/0` → `campus-notes-igw` | public-subnet |
| private-rt | VPC CIDR → local | private-subnet-a, private-subnet-b |

### Internet Gateway

**Name:** `campus-notes-igw`

The Internet Gateway is attached to `campus-notes-vpc` and provides internet connectivity for resources in the public subnet through the public route table.

The public route table contains a default route:

`0.0.0.0/0 → campus-notes-igw`

The private route table contains only the local VPC route. No NAT Gateway is configured.

### Network Traffic Flow

The intended network traffic flow is:

1. Students access the Campus Notes Board through the public IPv4 address of the EC2 instance.
2. Internet traffic reaches the VPC through the Internet Gateway.
3. The EC2 application server communicates with the RDS MySQL database over private VPC networking on TCP port 3306.
4. The EC2 instance accesses the private S3 bucket through its attached IAM role.
5. The RDS database remains inaccessible directly from the public internet.

## Questions

### 1. What is a VPC, and how is it different from a subnet?

A Virtual Private Cloud (VPC) is a logically isolated virtual network in AWS where cloud resources can be deployed and connected. A subnet is a smaller section of a VPC's IP address range that allows resources to be organized into different network segments.

### 2. What does /16 mean?

The `/16` represents the number of bits reserved for the network portion of an IPv4 address. A `10.42.0.0/16` network contains 65,536 total IPv4 addresses, providing a large address space that can be divided into smaller subnets.

### 3. What makes a subnet public?

A subnet is considered public when its associated route table contains a route to an Internet Gateway. Resources also need an appropriate public IP address and security group permissions to communicate directly with the internet.

### 4. Why do we separate public and private subnets?

We separate public and private subnets to improve security and control network access. Public subnets host resources that need direct internet connectivity, while private subnets protect sensitive resources such as databases from direct public access.

### 5. Why does RDS need a DB subnet group that covers two Availability Zones?

Amazon RDS requires a DB subnet group containing subnets in at least two different Availability Zones within the Region. This provides the network placement required for database deployment and supports Multi-AZ configurations for improved availability.

### 6. Why must the database not be publicly accessible?

The database stores application data and should not be directly exposed to the public internet. Keeping it private and allowing connections only from the application security group reduces the attack surface and prevents unauthorized direct database access.

## Security Considerations

The following security measures are part of the network design:

- The RDS database has public accessibility disabled.
- The `db-sg` security group allows MySQL traffic only on TCP port 3306 from `app-sg`.
- The EC2 application server uses `app-sg` to control network access.
- The S3 bucket remains private with Block Public Access enabled.
- The EC2 instance accesses S3 through an IAM role rather than permanent access keys.
- The private subnets do not have a direct Internet Gateway route.
- The main route table remains unchanged.

## Conclusion

The network design provides a structured and secure environment for deploying the Campus Notes Board application.

By using a custom VPC, separating public and private subnets, configuring route tables, and restricting database access through security groups, the infrastructure maintains controlled communication between application and database resources.

The design also provides a foundation for future improvements such as NAT Gateway connectivity, Multi-AZ database deployment, and additional application instances.

---

**Project:** Campus Notes Board  
**AWS Region:** `eu-north-1`  
**VPC:** `campus-notes-vpc`  
**VPC CIDR:** `10.42.0.0/16`