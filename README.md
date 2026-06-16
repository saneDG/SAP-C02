NOTE: These are my personal notes from my sudies to AWS SAP-C02 certification exam. Use as you wish but notes are created for me personally and might not make any sense to you.

#### <span style="color: #E67E22;">To Study Later (Keywords & Open Topics)</span>

- Cost optimization: Trusted Advisor (exact trigger scenarios)
- Cost optimization: Architect Spot Fleet integrations with EKS/ASGs (fault tolerance vs. cost)
- AWS service endpoints
- Deep dive into WAF
	- How to actually implement

Actions:
- Watch Cantrill demo lessons
- Complete all Cantrill practice exams
- Do couple of TD practice exams

#### Operational/Admin overhead
**Administrative overhead** is the cost of setting up the service. This includes setting up users, security configs, and all the monitoring tools available. These questions usually differentiated themselves by having answers that would show you could setup a given task faster with a tool like cloud formation or beanstalk. They usually have an element of security to them as well.

**Operational overhead** is the cost of the day-to-day operation of the service in question. The questions usually differentiated themselves by having answers that demonstrated knowing a service could potentially be expensive and you might be able to minimize the cost using an a different AWS service or sometimes not an AWS service at all. These require general knowledge of all the different services available. Reading this subreddit it seems these questions take the full breadth of AWS services into account. Mine was heavy with Lambda.

Both question styles try to trick you and test your deep knowledge by including multiple correct answers, but only one is distinguished and correct, usually by a descriptive indicator like "least effort" or "least expensive" in the question. If you don't see the indicator, i assume it's "least effort".

---

#### <span style="color: #27AE60;">Cost Optimization & Compute</span>

- **Savings Plans:** Compute flexibility.
- **Reserved Instances:** Instance lock.
- **Spread Placement Group:** ==Max 7 instances per AZ.==

---

#### <span style="color: #2980B9;">Systems Manager (SSM)</span>

![[Pasted image 20260527115109.png]]

- **Run Command:** Script execution without SSH.
- **Patch Manager:** OS patching schedules.
- **Parameter Store:** Configs and secrets.
- **State Manager:** Configuration baselines.
- **Session Manager:** Browser shell access.
- **Inventory:** Multi-account metadata collection.

---

#### <span style="color: dodgerblue;">📇 Identity and Federation</span>

##### Directory Service - Microsoft AD
- 2+ Domain Controllers
- Automatic patching and maintenance
- **AWS managed AD mode means that you have full directory running native Microsoft AD**
- On-prem AD and AWS Directory are separate things but they can be configured to trust each other

##### Directory Service - AD Connector
- Pair of directory endpoints running in AWS (ENIs in a VPC)
- Good for Proof of Concept (little admin overhead)
- Need network connection


--- 
### <span style="color: #8E44AD;">🕸️ Networking</span>

#### DNS

- A      -> IPv4
- AAAA   -> IPv6
- CNAME  -> NAME to another NAME
	- invalid for naked/apex
	- many AWS services use DNS name (ELBs)
	- with just CNAME pigg.fi => ELB would be invalid
	- solution is to use ALIAS!
- ALIAS  -> AWS resource
	- maps NAME to AWS resource
	- can be also used like CNAME
	- no charge for requests pointing at AWS resources
	- Implemented by AWS. Outside of DNS standard. Only in R53!
- MX     -> Mail
- TXT    -> text records
- NS     -> Name servers

#### <span style="color: #8E44AD;">Direct Connect (DX)</span>

- **1 Gbps DX:** 1000BASE-LX Transceiver
- **10 Gbps DX:** 10**G**BASE-LR Transceiver
- **100 Gbps DX:** 100**G**BASE-LR4
##### MACsec
- hop2hop
- Encrypt data between **on-prem** and **DX location**(not a AWS datacenter)
- NOT e2e
	- meaning: does not replace IPSEC over DX
- Terabit speed!

##### BGP Session + VLAN

VIF - Virtual Interface
- Allow you to run multiple L3 networks over the L2 DX
- PUBLIC, PRIVATE & TRANSIT
	- 50 pub & priv, 1 transit
	- 1 on each hosted connection

BGP - Border Gateway Protocol
- Is between the Customer DX router and the AWS DX Router
- can be extended to customer premises

VLAN
- Isolates

👇 Multiple VLANs. Allows isolated connections

![[Pasted image 20260529114051.png|184]]

Using those VLANS we create VIFs:

![[Pasted image 20260529114517.png|320]]

VIFs use VLAN and BGP session together

![[Pasted image 20260529115044.png|263]]

BGP sessions are authenticated via MD5


##### Private VIF
- 1 Private VIF = 1 VGW = 1 VPC
- AWS will advertise the VPC CIDR and the BGP Peer IPs (/30)
- YOU can advertise default or specific corp prefixes (MAX 100, HARD LIMIT)
- Can only connect to VPCs in the same region as the DX location (There is workaround)

Public VIF
- Access public services: elastic IPs, SNS, SQS, S3 etc
- Can access all public zone regions


--- 


#### VPC

##### VPC Routing Order
![[Pasted image 20260527133511.png|145]]

##### Interface Endpoint

- Provide private access to AWS public services
- Added to specific subnets - ENI - NOT HA
	- for HA add one subnet per AZ used in the VPC
- Uses PrivateLink
- Uses DNS (compared to Gateway Endpoint that uses prefix list)
	- e.g. `vpce-123-xyz.sns.us-east-1.vpce.amazonaws.com`

##### Gateway Endpoint

- Uses prefix list (compared to Interface Endpoint that uses DNS)
- Provide private access to S3 and DDB
- So this is used if you need private access to S3 or DDB. Use Interface Endpoint in other situations.
- HIGHLY AVAILABLE
	- Regional ... cant access cross-region services
- Endpoint policy is used to control what it can access
- Can be used to make S3 bucket private by allowing only access from gateway endpoint
- Only accessible from inside VPC


##### Internet Gateway
- Typically, if you need access to public service, you use Internet Gateway or Interface Endpoint within a VPC



--- 

#### Virtual Private Gateway (VGW)
The VGW is a managed, highly available gateway resource that you create and attach to your Amazon VPC. Think of it as the **entry point** for encrypted VPN traffic into your AWS cloud network.

#### Customer Gateway (CGW)
The term "Customer Gateway" can be slightly confusing because it refers to two distinct things depending on the context:
1. **The Physical Device:** The actual physical firewall or software router sitting in your on-premises data center (e.g., a Cisco ASA, Juniper, or Fortinet appliance).
2. **The AWS Resource:** The logical object you create inside the AWS Management Console to tell AWS about your physical device. When you create this resource, you provide AWS with your on-premises router's public IP address and its BGP ASN (Border Gateway Protocol Autonomous System Number) if you are using dynamic routing.
- **Where it lives:** Physically on-premises, but represented logically in AWS.
- **Responsibility:** It initiates the VPN tunnels out to AWS and handles the encryption/decryption on your side of the network.

#### Transit Gateway (TGW)
- This is the gateway which can be used to reduce complexity significantly!!
- Central hub. Supports *transitive routing*. Peering across accounts/regions.

#### NAT Gateway (Network Address Translation)
- **Function:** Allows private subnets to access the internet (outbound only). Must live in a **Public Subnet**.
- **HA Rule:** Redundant only _within its own AZ_. For multi-AZ fault tolerance, you need one NATGW per AZ + separate route tables.
- **Cost Pivot:** Billed per GB processed. If traffic is going to S3/DynamoDB, bypass NATGW and use a **Gateway Endpoint** to save money.
- **Security:** Does _not_ support Security Groups (use NACLs instead).


#### <span style="color: tomato;">🤝 NACL and Security Group</span>

##### NACL

- Associated ONLY with SUBNETS
	- No logical resources
- Each subnet can have ONE NACL (default or custom)
     AND
- NACL can be assigned to MANY SUBNETS
- Connections WITHIN SUBNETS ARE NOT IMPACTED
- STATELESS
	- Need to add rule both INBOUND and OUTBOUND
- NACL offer both explicit allow and explicit deny
- Rules are processed in order
- Default NACL when creating a VPC: ALL TRAFFIC ALLOWED 
     AND
- Custom NACL DELY ALL by default
- Only IP/CIDR, Ports & Protocol - No logical resources


##### Security Groups (SG) (VPC)

- STATEFUL
- NO EXPLICIT DENY ... only ALLOW or Implicit DENY
	- For example when you allow inbound HTTPS `0.0.0.0/0`, no rule can override this range of inbound requests more specifically. You can't for example block inbound HTTPS from any single IP at this point.
- Can't block specific bad actors
- Supports IP/CIDR AND logical resources
	- Like other SG's and ITSELF
	- Using self reference IP changes are automatically handled!! Useful for example when using ASG
- Attached to ENI's, not to instances (even if the UI shows it this way)


SG & NACL notes:
Think of SG as something that surrounds a Network Interface. NACL surrounds a subnet.

### <span style="color: orange;">📍 Local Zones</span>

AWS Local Zones are a type of infrastructure deployment that places compute, storage, database, and other select AWS services close to large population and industry centers.



### <span style="color: GreenYellow;">🚛 Migration</span>


#### 6R's of Cloud Migration
- Rehosting
- Replatforming
- Repurchasing
- Refactoring/Re-architecting
- Retire
- Retain

#### Application Discovery Service

AWS Application Discovery Service helps enterprise customers plan migration projects by gathering information about their on-premises data centers.

- Discover on-premises infrastructure
	- VM cpu, memory, MAC addresses, utilisation etc.
- Agentless (uses Application Discovery Agentless Connector)
	- Unable to measure outside of the instance
- Agent based mode
	- network usage, processes, usage, performance, dependencies between servers (network activity)
- Integrates with AWS Migration Hub and Amazon Athena(Ad-hoc analysis)

#### Server Migration Service (SMS)

AWS Server Migration Service (SMS) is an agentless service which makes it easier and faster for you to migrate thousands of on-premises workloads to AWS. AWS SMS allows you to automate, schedule, and track incremental replications of live server volumes, making it easier for you to coordinate large-scale server migrations.

- Migrate whole VM's (OS, Data, Apps ... migrated as is)
- Agentless ... uses connector which runs on-premises
- Integrates with VMWare, Hyper-V and AzureVM only
- Incremental replication of live volumes
- Orchestrate multi-server migrations
- Creates AMIs which can be used to create EC2 instances
- Integrates with AWS Migration Hub

#### Database Migration Service (DMS)

- Runs using replication instance
- Source and Destination endpoints point at Source and target databases
	- One endpoint must be on AWS
- Jobs can be
	- Full load
		- one off migration of all data
	- Full load + CDC (Change Data Capture)
		- for ongoing replication which captures changes on the source
		- when the migration is complete, the changes that happened while the migration, the changes are also applied to the target database
	- CDC Only
		- Replicates only data changes
- No schema conversion (but that can be done with SCT. more below)

##### Schema Conversion Tool (SCT)

- Only used when converting **one database engine to another**
- including DB -> S3 (Migrations using DMS)
- SCT is not used when migrating between DB's of the same type!
	- for example NOT USED: on-prem MySQL -> RDS MySQL (same engine)
- EXAMPLE: on-prem MSSQL -> RDS MySQL
- EXAMPLE: on-prem ORACLE -> Aurora
- EXAMPLE: on-prem MySQL - Snowball (Generic file format)

##### DMS & Snowball
- Large migrations might be multi TB in size. Moving data over networks takes time and consumes capacity.
- DMS can utilise snowball
Steps:
1. Use SCT to extract data locally and move to a snowball device'
2. Ship the device back. AWS loads the data to S3
3. DMS migrates from S3 into the target store
4. Change Data Capture (CDC) can capture changes, and via S3 intermediary they are also written to the target database

#### Migration Hub
- Single location for all migration workloads - SMS/DMS etc

#### Physical Migration
- Snowball
	- Only storage!
	- 10TB to 10PB
	- Multiple devices, multiple locations
- Snowball Edge
	- Both **Storage** and **Compute**!
- Snowmobile
	- Not economical for multiple site!!
	- Ideal for single location when 10PB+ is required
	- up to 100PB
	- Literally a truck
	- Not available everywhere


#### Storage Gateway
- Migrations, Extensions, Storage Tiering, DR and replacement of backup systems
- For exam, if you see mention of: 
	- Volumes -> default to Volume Mode
	- Tapes -> Tape mode
	- Files -> File mode
**Volume Gateway**
- All stored LOCALLY
- Great for **full disc backups**
- Assist with disaster recovery ... create EBS volumes in AWS
- DOES NOT IMPROVE DATACENTER CAPACITY
	- main copy of data is stored on the gateway
**Volume Cached**
- Data is in **AWS S3!!!**
	- AWS managed. You can't preview the data yourself.
- Storage appears on-premises but its actually in AWS
- Data is stored in AWS but Cached on-premises
- Capacity extension
**Tape Gateway (VTL (Virtual Tape Library) mode)**
- Storage gateway in VTL mode allows the product to replace a tape based backup solution with one which uses S3 and Glacier rather than physical tape media.
- Large backups -> TAPE
**File Gateway**
- Bridges file storage and S3
- Mount Points (shares) available via NFS or SMB
- Files stored into a mount point, are visible as objects in a S3 bucket
	- File shares and the bucket are linked together
- Read and Write Caching ensure LAN-like performance
- Primary data is held in S3!!!
- you can integrate with other AWS services
- supports S3 object lifecycle Management
	- for example: Standard -> Standard IA -> Glacier
- You can share files between multiple on-premises servers
Drawbacks with multiple on-prem servers:
- Use **NotifyWhenUploaded** API to notify other gateways when objects are changed
	- Not handled by default
- File GW does not support Object locking - use read only mode on other shares or tightly control file access


#### AWS DataSync
- AWS DataSync is a product which can orchestrate the movement of **large scale data** (amounts or files) from on-premises NAS/SAN into AWS or vice-versa
- Used for HUGE SCALE
- Keeps metadata
- built in data validation
- scalable
- bandwidth limiters
- compression and encryption
- Automatic recovery from transit errors
- pay as you use (per GB data moved)
- **DataSync agent** runs on VM platform (e.g. VMWare) and communicates with AWS **DataSync Endpoint**
	- Agent runs on-premises, used to read/write using NFS or SMB
- DataSync Endpoint communicates with NFS, SMB, FSx, S3, EFS...
- Schedule can be set to ensure the transfer of data occurs during or avoiding specific time periods
- KEYWORDS: Incremental transfer, bi-directional transfer, scheduled transfer


--- 


### <span style="color: tomato;">💾 Storage</span>

##### Instance Store Volumes
- SUPER HIGH throughput
	- D3 4,6GB/s
	- I3 16 GB/s
- More IOPS and throughput vs EBS
- Millions IOPS
- TEMPORARY block-level storage
- Physically attached
- Local to EC2
- Can ONLY add at EC2 LAUCH
- Lost on instance move, resize or hardware failure
- TEMPORARY, TEMPORARY, TEMPORARY, NOT PERSISTENT



![[Pasted image 20260527152839.png|358]]

![[Pasted image 20260529132156.png|359]]

##### EBS
- No cross-AZ!!
- Can create volume snapshot to S3
	- and use that to recover AZ failure
	- also possible to use cross-region with S3

##### EFS
- POSIX
- Linux only!!!
- EFS is an implementation of NFSv4
- Mounted in Linux
- Shared between many EC2 instances
- Accessed via **mount targets**
- Most use cases use General Purpose
	- Max I/O in rare cases
- Bursting an Provisioned modes
- Standard and Infrequent access

##### FSx
- For Windows!!
- Fully managed
	- you get file shares
- Single or multi-AZ modes within a VPC
- Can be used within AWS, or from on-premises environments via VPN or Direct Connect
- Can integrate to on-prem AD. No need to implement AD yourself if exists.
EXAM KEYWORDS:
- **VSS** (Windows feature, User-driven restore, restore without admin)
- Native filesystem accessible over **SMB**
- Windows permission model
- Supports DFS (scale-out file share structure)
- Integrates with **DS** and your own directory (Active Directory, Directory Service)

##### FSx for Lustre
- Designed for HPC - LINUX Clients (POSIX)
- ML, Big Data, Financial Modeling
- If you see any mention on high performance, big data, ML, Linux, POSIX etc. Choose FSx for Lustre!

#### <span style="color: palegreen;">🪣 S3</span>

##### S3 Encryption
Server side encryption
- SSE-C   - Customer provided keys
	- You provide the key and the plaintext object
	- S3 manages the encryption
	- You have still trust AWS at some level. For example to discard the key
	- If stricter security practices are needed: use Client Side Encryption
- SSE-S3  - AWS managed keys, (This is the default)
	- AES256
	- You provide the plaintext object
	- AWS creates, manages, rotates the keys
- SSE-KMS - Keys stored in AWS KMS
	- This is the most secure if needed in medical apps etc.
	- KMS key created, managed by you
	- Fine grained control
	- Even S3 admin does not have control to encryption process by default
		- If not allowed in KMS

![[Pasted image 20260529122538.png|404]]



##### S3 Replication
- CRR - Cross Region Replication
- SRR - Same Region Replication
- RTC - Replication Time Control
	- Optional if you need "15 minute replication"
- By default NOT RETROACTIVE & versioning needs to be on
- One way replication (or bi-directional)
- System events or Glacier cant be replicated
	- user events are replicated
- No deletes (but this can be added - **DeleteMarkerReplication**)


S3 REPLICATION EDGE CASE:
**Cross-Account S3 + CRR:** Partner bucket policy uploads break replication due to object ownership issues. Pivot to **IAM Role (AssumeRole)** so the hosting account owns the objects and CRR succeeds.

##### S3 "Requester pays" feature
You must authenticate all requests involving Requester Pays buckets. The request authentication enables Amazon S3 to identify and charge the requester for their use of the Requester Pays bucket. After you configure a bucket to be a Requester Pays bucket, requesters must include x-amz-request-payer in their requests either in the header, for POST, GET and HEAD requests, or as a parameter in a REST request to show that they understand that they will be charged for the request and the data download.


##### S3 Access Points
Amazon S3 Access Points, a feature of S3, simplifies managing data access at scale for applications using shared data sets on S3. Access points are unique hostnames that customers create to enforce distinct permissions and network controls for any request made through the access point.
- Simplify managing access to S3 Buckets/Objects
- Rather than 1 bucket w/ 1 Bucket Policy ... create many access points
- ... each with different policies
- ... each with different network access controls
- Each access point has its own endpoint address
- CLI command for creating an access point: `aws s3control create-access-point`
EXAMPLE:
- Create access point for multiple different type of users of the bucket
- create access point policy for each access point
	- restricts access to identities certain prefix, tags or actions
- Each access point has unique DNS address for network access
- VPC origin
	- Access point can be also configured for access via VPC
	- Requires VPC endpoint
	- Access via this route can be enforced by endpoint policies
- You need matching permissions with the Access Point and bucket
	- So if certain group has been granted access via Access Point Policy, the same access needs to be via the Bucket Policy
	- You can also do delegation
		- On the bucket policy grant all access via Access point
			- More granular access control in Access Point Policy

#### Amazon Macie
- Data Security and Privacy service
- Discover, monitor and protect the data stored in S3 buckets
- Automated discovery of data i.e PII, PHI, Finance
- Managed data identifiers - Built-in ML/patterns
- Custom Data identifiers - Regex based
- Integrates with Security Hub & 'finding events' to EventBridge
**Identifiers**
- Managed data identifiers - maintained by AWS
	- Growing list of common sensitive data types
	- Credentials, finance, PII...
- Custom data identifiers - created by YOU
	- regex
	- Maximum Match Distance - how close keywords are to regex pattern
	- Ignore words can be defined
Findings
- Policy findings
	- For example detects when S3 BlockPublicAccessDisabled is enabled
- Sensitive data findings
	- detects sensitive data in the files in S3 bucket files


#### <span style="color: palegreen;">👮🏻‍♂️ Permissions & Accounts</span>

REMEMBER THESE
1. Policies have a default **implicit deny**
2. An **explicit deny overrides an explicit allow**
3. It does not matter whether an ALLOW or DENY comes from an inline or managed policy

##### **SCP** - Service Control Policy
- **Management account is special** SCP has no a effect
- A feature of AWS Organizations which allow restrictions to be placed on MEMBER accounts in the form of **boundaries**.
	- Remember that SCPs are just boundaries
- SCPs can be applied to the organization, to OU's or to individual accounts.
- Member accounts can be affected, the MANAGEMENT account cannot.
- SCPs DON'T GIVE permission - they just control what an account CAN and CANNOT grant via identity policies.
- Example: Control what size of EC2 instance can be used.

Memorize this:
![[Pasted image 20260529124157.png|446]]

##### STS - Security Token Service


##### Roles
- Roles can be assumed by many identities


EXAMPLE SCENARIO: Revoking temporary credentials if leaked. Identities received credentials based on the roles **permission policy**
What does not work:
- Changing trust policy has **no** impact on existing credentials
- Changing the permissions policy would work BUT it impacts ALL CREDENTIALS for all users
What works:
- Update the permissions policy with a AWSRevokeOlderSessions inline DENY for any sessions older than NOW

#### Databases

##### <span style="color: aqua;">RDS</span>

- Database SERVER as a Service
- MySQL, MariaDB, PostgreSQL, Oracle, MS SQL Server
- Amazon Aurora is different product!!

##### <span style="color: aquamarine;">Amazon Aurora (RDS)</span>

- RDS but AWS managed
- Uses a "Cluster"
- Single primary instance + 0 or more replicas
- uses shared cluster volumes (faster)
- up to 15 replicas
- Only SSD based
- Uses endpoints
	- Cluster endpoint
	- Reader endpoint
- No free tier option
- Backtrack can be used, allows in-place rewinds
- Fast clones - copy-on-write

![[Pasted image 20260529165146.png|285]]

###### Aurora Serverless

- Set MIN and MAX ACU (Aurora Capacity Units) and aurora scales based on that
- can go to 0 and be paused
- same resilience as Aurora (6 copies across AZs
Use cases:
- Infrequent access
- New applications, possible changes coming in the future
- Variable workloads
- Development and test DB
- Multi tenant applications

###### Aurora Multi-Master
- Default Aurora mode is Single Master
	- failover takes time
- In Multi-Master mode ALL instances are R/W

##### <span style="color: thistle;">Amazon Athena</span>

- Serverless Interactive Querying Service
- Ad-hoc queries on data, pay only data consumed
- Schema-on-read - table-like translation
- Original data never changes - remains on S3
- Schema translates data => relational-like when read
- Output can be send to other services

EXAMPLE
1. source data (ANY FORMAT), Read only, S3, Data in S3 is fixed
2. Create schema, table-like structure
	1. define how convert the data
	2. tables are created in advance in a data catalog
	3. it allows SQL-like queries on data without transforming the source data
3. Data is streamed through schema
4. Output can be send to visualization tools

Athena is good for (keywords for exam):
- Occasional / Ad-hoc queries on data in S3
- Serverless querying - Cost conscious
- Querying AWS logs - VPC Flow logs, CloudTrail, ELB logs, cost reports etc...
- Glue data calatog & Web Server logs
- Athena Federated Query .. other data sources
	- Usually if question mentions: SQL, noSQL or any specific DB product, Athena is usually not the answer!!

#### Caching

##### ElastiCache (Kattotermi)

- In-memory database .. high performance
- Managed Redis or Memcached .. as service
- For READ HEAVY workloads
- reduce database workloads (expensive)
- Can be used to store **Session Data** (Stateless Servers)
- Requires APPLICATION CODE CHANGES!!!
Using ElastiCache as session state store. Makes application fault tolerant!!:
![[Pasted image 20260605093118.png|395]]
Using ElastiCache as in-memory cache (the common way of using it):
![[Pasted image 20260605093244.png|397]]

**Memcached**
- Simple data structures
- No replication, single AZ
- No backups
- Multi threaded

**Redis**
- Advanced data structures
- e.g. game leaderboard
- Multi AZ, true replication across instances
- Backup and restore
- Transactions

#### Data Analytics

##### Kinesis
- Scalable streaming service
- Producers send data into a kinesis stream
- Streams can scale from low to near infinite data rates
- public & HA
- store data for 24h, can be increased to 365 (increased cost)
- multiple consumers access data
Kinesis Architecture
- Data stored in Kinesis Data Record (1MB)
- If need to store data long term, use Firehose to store data for example in S3
- Used in ingestion, Analytics, Monitoring, **APP CLICKS**

### Monitoring & Logging

#### CloudWatch

- Keyword: METRICS
- For example EC2 supports CW automatically but only with metrics visible from outside
	- If you need richer metrics (Like CPU and other performance metrics) you need to install **Agent**
- On-prem integration via agents or API
- Application integration via Agent or API
- **Alarms**: react to metrics
	- you can configure one or more actions like SNS or some ASG action etc.
- Resolution: Standard (60s granularity) ... high (1s granularity)
- Retention: 
	- sub 60s retained for 3h
	- 60s (1m) retained for 15 days
	- 300s (5m) retained for 63 days
	- 3600s (1h) retained for 455 days
	- as data ages, its aggregated and stored for longer with less resolution

##### CloudWatch Logs

- **CWAgent** - If you want logging from system or custom application
- Regional service
- By default indefinite storage but this can be configured
- You can configure **Subscription Filter** to stream near-real-time logs
	- If you need realtime, use lambda or Kinesis Data Stream
- If you need metric as output, create **Metric Filter** in CW
- supports:
	- VPC Flow Logs
	- CloudTrail (Account events and AWS API calls)
	- Elastic Beanstalk
	- ECS
	- API GW
	- Lambda
	- Route53 (Log DNS requests)

##### CloudWatch Events & EventBridge
- If x happens, or at y times, do z
- EventBridge is newer version of CloudWatch Events
	- You should use EventBridge by default for new apps
- Can define event rule pattern or schedule rule
	- then point to target for example to invoke lambda
- Data in the event can be used in the target

#### X-RAY

- Scans through infrastructure
Supported resources:
- **EC2** - X-Ray Agent
- **ECS** - **Agent in tasks**
- **Lambda** - enable option
- **Beanstalk** - agent preinstalled
- **API Gateway** - per stage option
- **SNS** & **SQS**
- Requires **IAM PERMISSIONS**

#### CoudTrail

- Logs API calls and account events as **CloudTrail Event**
- NO REAL TIME LOGGING!!!
	- 15min delay
- 90 days stored by default
- Regional service
	- You can add trail for **one region** or **all regions**
- **Management events**
	- Implemented by default
	- example: creating EC2 instance or any other resource
- **Data events**
	- not enabled by default because data amount would be massive if every data event would be logged!
	- example: object loaded from S3 or deleted or any other resource data event
- Store data in **S3** or **CloudWatch Logs**
- You can create Organizational trail: Create trail from Management Account of the organization -> It stores all events in that organization

### <span style="color: wheat;">🛡️ Security</span>

#### <span style="color: red;">GuardDuty</span>
- Continuous security monitoring service
- Analyses supported data sources
	- ... + AI/ML + threat intelligence feeds
- Identifies unexpected and unauthorised activity
- notify or event-driven protection/remediation
- supports multiple accounts (MASTER and MEMBER)

#### AWS Shield
- DDoS
- WAF integration

##### Amazon Inspector
- Scans EC2 instances & instance OS
- ... also containers
- Network Assessment (Agentless)
- Network and host assessment (Agent)
- Rules packages determine what is checked
KEYWORDS:
- CVE - Common Vulnerabilities and Exposures!!!
- CIS - Center for Internet Security Benchmarks!!!
- Security best practices

### Cost Management

**On-Demand**:                 most expensive
**Savings Plan(Reserved)**:    up to 72% cheaper than on-demand
**Spot**:                      up to 90% cheaper than on-demand, trmnted 2min warning

_Note for the exam:_ **Savings Plans** are the modern replacement for RIs. Instead of committing to specific instance types (like `m5.large`), you commit to a dollar amount per hour (e.g., $10/hour). Savings Plans are much more flexible, applying across different instance families and even AWS Fargate and Lambda.


### <span style="color: gold;">🌎 Caching, Delivery and Edge</span>

#### <span style="color: fuchsia;">CloudFront</span>

**Origin** - The source location of your content
- **S3 Origin** (bucket)
- **Custom Origin** (Anything that has public IPv4 address)
**Edge Location** - Local cache of your data
**Regional Edge Cache** - Larger version of an Edge Location
- Provides another layer of caching

Integrates with **ACM** for HTTPS
ONLY DOWNLOAD CACHING
- Uploads direct to origin - NO CACHING

##### Distribution

**Distribution** - The 'configuration' unit of CloudFront
At Distribution level, you configure:
- WAF web ACL
- Alternate domain name
- Custom SSL certificate
- the configuration deployed to the **edge locations**
- security policy
	- If you select more recent security policy, then potentially you can prevent clients with older browser from accessing your distribution
- Supported HTTP versions
- Logging on/off
- IPv6 on/off

If you want to use alternate domain name and use that with https then you need to use a custom SSL certificate. Thats defined at the distribution level. Uses **ACM**.

##### Behaviour

**Behaviours** - Single **distribution** can have multiple **behaviours** which are configured with a path pattern
- Uses path pattern
- Default behaviour is (\*)

At Behaviour level you can configure:
- Select origin and origin groups (THE MOST IMPORTANT ONE)
- Viewer protocol policy (HTTP and HTTPS, Redirect HTTP to HTTPS, HTTPS only)
- Allowed HTTP methods (GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE)
- to restrict viewer access (important!)
	- If restricted, you must use CloudFront signed URLs or signed cookies!!
	- If restricted, you must configure trusted Key groups (This is new way of doing this) or Trusted signer!!!
- Cache key and origin requests
- Lambda @edge is also configured at **behaviour** level!!
- Set to compress objects automatically

##### Cloudfront TTL and Invalidations

![[Pasted image 20260603130833.png|434]]

- Responses from Cloudfront origin:
	- **304 Not Modified** - The cache is valid and object is delivered from edge location
	- **200 OK** - Cache does not have valid content, new content is delivered and replaced in the edge location

**TTL**
- Default TTL (behaviour) is 24h
- You can set minimum TTL and maximum TTL values
	- min and max acts as limiters for any per-object TTL values!!
TTL headers:
- Cache-Control **max-age**(or "**s-maxage**") (seconds)
	- after the number of seconds specified, the object will be viewed as expired
- **Expires** (date & time)
	- Same as "max-age" but defined as datetime
- Can be set as Custom Origin or S3 (Via object metadata)!!!
Cache Invalidations
- Cache Invalidations are performed on a **distribution**!!
	- Meaning: Applies to all edge locations ... **takes time**
- Cache Invalidation costs money!!
	- If you need to constantly invalidate or update the cached objects: use VERSION NAMES in the files (img_v1.jpg, img_v2.jpg, ...)

##### Cloudfront SSL & SNI
- SSL supported by default ... *\*.cloudfront.net* cert
- Most of the time you want to use your own custom name with your own CloudFront distribution
	- That's called: **Alternate Domain Name**
- Generate or import certificate in ACM (in us-east-1)!!
- Two SSL Connections: 
	- Viewer -> CloudFront (Viewer Protocol)
	- CloudFront -> Origin (Origin Protocol)
- BOTH OF THE CONNECTIONS NEED VALID **PUBLIC** CERTIFICATES (and intermediate certs)
	- Self signed certificates WILL NOT WORK with CloudFront!!
**SSL/SNI (Server Name Indication)**
- SNI is a TLS extension, allowing host to be included
- You don't need to assign certificates to S3 Origins because its handled natively for you.
- If you need to support pre-historic browsers (pre ~2003) to non SNI, you need dedicated IP at each CF Edge Location.
	- dedicated IP cost is 600$/month
	- SNI-mode is free (but not supported by old browsers)

Rules about the certificates at Viewer and Origin sides:
**Viewer**
- Any public certificate, must match the name of the cloudfront DISTRIBUTION its applied to
	- **So if you add a custom domain name, the DNS needs to point at CloudFront, and the certificate needs to match the DNS name that you are using!!!!!**
	- The certificate here must match the **Custom Alternate Domain Name (CNAME)** that the user typed into their browser (e.g., `app.yourdomain.com`).
- **Purpose:** To prove to the user's browser that they are genuinely communicating with `app.yourdomain.com`.
**Origin**
- Any public certificate, must match the name of the cloudfront DISTRIBUTION its applied to (same as viewer)
- S3: No certificates (actually you cant even change the certificate of S3)
- ALB: Needs publicly trusted certificate.
	- You can use your own cert from CA ...
	- or you can use ACM to generate and manage on your behalf
- Edge case - Custom Origin (EC2 or on-prem)
	- ACM does not support EC2!!
	- You need to use your own certificate from trusted authority (DIFFERENT THING THAN SELF SIGNED, NO SELF SIGNED)
	- CloudFront DOES NOT SUPPORT SELF SIGNED CERTIFICATE

##### Caching performance and optimisation
Forwarding:
- By default query parameters are not forwarded:
	- `.../item.jpg&color=COLOR&size=SIZE` ->> `.../item.jpg`
- If you forward all and dont cache:
	- `.../item.jpg&color=COLOR&size=SIZE` ->> `.../item.jpg&color=COLOR&size=SIZE`
	- This is wrong because every request is **cache miss**
- If you forward everything and use cache whitelist (see image)
![[Pasted image 20260604104402.png|418]]

##### CloudFront Security - OAC (old OAI) & Custom Origins
- Origin Access identities (OAI) - for S3 Origins
- Custom Headers - For Custom Origins
- IP Based FW Blocks - For Custom Origins.

##### OAC (old OAI) (Only for S3 origins)
- OAI is a type of identity
- It can be associated with CloudFront Distributions
- CloudFront becomes that OAI
- That OAI can be used in S3 Bucket Policies
- DENY all BUT one or more OAI's
- When implementing this, you need to configure policy for S3 bucket to allow s3:getObject from CloudFront distribution (referenced by ARN in the policy)

> Here is exactly how you can expect to see OAI tested on the exam:
> 
> - **As a Distractor:** OAI will often appear as a plausible but incorrect option. If a question asks for the "most secure" or "most modern" way to restrict S3 access to CloudFront, and both OAI and OAC are options, OAC is almost certainly the correct choice.
>     
> - **The KMS Encryption Trap:** This is a classic SAP-C02 scenario. A question will describe an existing architecture using CloudFront, OAI, and S3. The company's new compliance policy requires all S3 data to be encrypted with customer-managed KMS keys (SSE-KMS). The question will ask how to implement this. If you don't know that OAI cannot support KMS, you might pick a wrong answer. The correct solution will involve upgrading the distribution from OAI to OAC.
>     
> - **Migration Strategies:** You may be asked how to migrate a production system from OAI to OAC with zero downtime. The correct approach involves updating the S3 bucket policy to allow _both_ the OAI and the new OAC IAM service principal temporarily, updating the CloudFront distribution to use OAC, and then finally removing the OAI from the bucket policy.

##### CloudFront - Custom Origins

Custom Headers
- You configure CloudFront to add custom header that is sent from edge location to the custom origin
- origin is configured to require this header, otherwise it wont service requests
- because entire stream is using HTTPS, nobody can oversee the headers and fake them
IP Based Firewall Block
- traditional
- the IP ranges of the edge locations are easy to determine
- we can configure firewall to allow connections only from the edge locations and block everything else
- For example the request directly from client to origin would be blocked

Both of these methods can be used **at the same time** for **custom origins**

##### Security - Private Distributions, Signed URL and Signed Cookies
- By default CF is public
- private ... requests require **signed cookie** or **signed url**
- OLD WAY: CF Key is created by account root user
- OLD WAY: The account is added as trusted signer
- NEW WAY: TRUSTED KEY GROUP(S) added

**Signed URL**
- Provides access to one object only
- Use URL's if your client does not support cookies

**Signed Cookie**
- Provide access to groups of objects
- Use for group of files/all files of a type
	- e.g. all cats gifs
- Or if maintaining application URL's is important

##### <span style="color: cyan;">🔔 CloudFront - Geo Restriction IMPORTANT</span>
CloudFront Geo Restriction (CloudFront builtin feature)
- CF - whitelist or blacklist - COUNTRY ONLY!!!
- CF - GeoIP Database 99,8%+ Accurate
- CF - Applies to **entire distribution**
3rd Party Geolocation
- Completely customisable
- restrict based on user, browser or login state of an application
- We need some form of application server
- CF is configured to be private
	- that means that any access will require a signed cookie or URL
- We could utilise Geo Location Database or anything else
	- like username, or any other attribute
	- you can go beyond location. you can do whatever you want.

##### Field-Level Encryption
Field-Level encryption allows CloudFront to encrypt certain sensitive data at the edge using a public key, ensuring its protection through all levels of an application stack. Only the corresponding private key can decrypt the data, meaning you have complete control over who has access.

##### CF - Lambda@Edge
- You can run lightweight lambda at edge location
- adjust data between viewer and origin
- supports Node.js and python
- run in the AWS Public Space (not VPC)
- Lambda layers not supported
- Different limits vs normal lambda functions
You can run lambda at 4 different stages of the stream:
![[Pasted image 20260605092044.png|416]]
- Use cases:
	- A/B testing - Viewer Request
	- Migration between S3 Origins - Origin Request
	- Different Objects Based on Device - Origin Request
	- Content By Country - Origin Request

--- 

### Everything else
##### <span style="color: red;">AWS Glue</span>

- AWS Glue is a fully managed extract, transform, and load (ETL) service
- Serverless

AWS Glue - Data Catalog
- Persistent metadata about data sources in region

Keywords for the exam:
- If datapipeline and AWS Glue is both in same question, look for specs like "serverless", "Ad-hoc" or "cost effective", in that case select **Glue**
- datapipeline uses EMR

##### Kinesis Data Stream
- This is the default I guess?
- Real-time (~200ms)
##### Kinesis Firehose
- Connects to Kinesis Data Stream ^
- Fully managed service
- 60s delay, no real time
- valid destinations:
	- HTTP
	- splunk
	- Redshift
	- ElasticSearch
	- S3 Bucket
##### Amazon Kinesis Data Analytics
- real-time processing of data
- uses SQL
- Ingests from Kinesis Data Streams or Firehose
- used for example real-time leaderboards for games, e-sports, elections

##### Launch Configuration (ASG)
- Define what is launched by an ASG
- ASG can have max of 1 LC
##### Launch Template (ASG)
- Define what is launched by ASG OR **manually** via launch instance
- ASG can have max of 1 LT


#### AWS Config
AWS Config is a service which records the configuration of resources over time (configuration items) into configuration histories.

All the information is stored regionally in an S3 config bucket.

AWS Config is capable of checking for compliance .. and generating notifications and events based on compliance.

### 💈 Data Analytics

MapReduce 101
- Data Analysis Architecture
- Two phases - MAP and REDUCE
- Optional phases - Combine & Partition

MapReduce flow:
1. takes data as input (for example all text in the every book)
2. data is separated into **splits**
	- splits are made based on split size
3. each split is assigned into a **mapper**
4. Mapping outputs are shuffled to consolidated records 
	- for example same words are grouped together
5. Mapped results are **reduced**
6. In the end data is **recombined** into final output

In the end the output is reduced up to millions in size than input

HDFS - Hadoop File System
- Available in every MapReduce system. widely used.

#### <span style="color: burlywood;">EMR - Elastic Map Reduce</span>

- input -> EMR -> output
- Core nodes manage HDFS storage
- EMRFS is resilent file system supported natively within EMR
	- resilient to core node failures

![[Pasted image 20260529151632.png|329]]

#### <span style="color: red;">Redshift</span>

- Petabyte scale DATA WAREHOUSE
- OLAP (column based)
	- not OLTP (row/transaction)
- NOT SERVERLESS
- Pay as you use
- Runs only single AZ
	- you can backup to S3 tho
	- can select to save backups to multiple AZ and regions


#### <span style="color: tomato;">AWS Batch</span>

- Jobs will be added to queues
- Loads and uses container image

Lambda vs Batch
- Use for example when lambda 15min limit is an issue
- Lambda limited disc space
- Batch is not serverless
- Lambda has limited runtimes
- Batch has no resource limit

Managed vs Unmanaged
- Managed = AWS manages things (least admin overhead)
- Unmanaged = You manage everything (most control)


#### <span style="color: green;">Amazon QuickSight</span>

- Business Analytics / BI service
- In the exam, keywords: **dashboard or visualization**

#### AWS Device Farm

- managed Web and Mobile application testing
- test on real fleet of devices

--- 

### Serverless

#### API Gateway
API Gateway Endpoint types:
- Edge-Optimized - Routed to the nearest CloudFront POP (Point of Presence)
- Regional - Clients in the same region
- Private - Only accessible within a VPC
API Gateway Stages:
- for example PROD and DEV
- You can rollback deployed stages
- Traffic can be distributed between the stages
	- For example deploy v2 to DEV stage and sent certain percentage of the traffic to DEV
API Gateway errors:
- 4XX - Client error
- 5XX - Server error
- 400 Bad request - Generic
- 403 Access denied - Auth denied... WAF filtered
- 429 - API Gateway throttle
- 502 - Bad gateway - Bad output returned by lambda
- 503 - Service unavailable - endpoint offline? major service issues
- 504 - Integration failure - Timeout 29s limit
API Gateway caching:
- Cache is defined per stage within API Gateway
- Cache can be encrypted
API Gateway Integrations:
- MOCK - Used for testing only, no backend
- HTTP - Backend HTTP endpoint (uses Mapping Template)
- HTTP Proxy - pass through unmodified (NO Mapping Template)
- AWS - Lets an API expose AWS service actions (uses Mapping Template)
- AWS_PROXY (LAMBDA) - Low admin overhead Lambda endpoint (NO Mapping Template)
API Gateway Mapping Templates
- Used for non proxy integrations
- Modify or rename parameters
- Modify the body or headers
- filtering
- uses VTL (Velocity Template Language)


#### State Machine
- Max duration 1 year
- serverless

#### Elastic Transcoder (ET) &  Elemental Media Convert (MC)
- File-based **video transcoding service**
- MediaConvert is sort of Elastic Transcoder v2
- Serverless - Pay for resources used
- Add jobs to pipelines (ET) or Queues (MC)
- File loaded from S3, processed, stored on S3
- MC supports EventBridge


#### AWS IOT
- AWS IOT Core is a suite of products, not one thing
- Provisioning, Updates & Control
- Unreliable links - device shadows
AWS IOT Greengrass
- Extends some AWS services to the edge
- Some compute, messaging



--- 

### 🧠 AI
#### Amazon Kendra
**Amazon** **Kendra** is an intelligent search service powered by machine learning (ML).
#### Amazon Textract
**Extremely common**. Usually paired with S3 and Lambda in scenarios where a company is digitizing thousands of physical documents, invoices, or forms and needs to extract the text and structural data automatically.
#### Amazon Comprehend  [Previous Lesson](https://learn.cantrill.io/courses/895720/lectures/23375911)
Often chained together with Textract. Once Textract pulls the text from a document, Comprehend is used to find PII (Personally Identifiable Information) so it can be redacted, or to perform sentiment analysis on customer reviews.
#### Amazon Rekognition
- Deep learning image and videos
- can integrate with Kinesis Video Streams to analyze live video stream
Look for scenarios involving security cameras, automating video moderation, or identifying specific objects/faces in user-uploaded images.
#### Amazon Lex
The engine behind Alexa. Used when a scenario requires building conversational chatbots for customer service or call center automation (often integrated with Amazon Connect).
#### Amazon Translate
Translates
#### Amazon Transcribe
Speech recognition
- text indexing, meeting notes, captions, call analysis
#### Amazon Forecast
- Forecasting for time-series data
- Import historical & related data ... understands whats normal
- Outputs forecast and forecast explainability
#### Amazon Fraud Detector
Amazon Fraud Detector is a fully managed fraud detection service that automates the detection of potentially fraudulent activities online. These activities include unauthorized transactions and the creation of fake accounts. Amazon Fraud Detector works by using machine learning to analyze your data.
#### Amazon SageMaker
Amazon SageMaker is a fully managed machine learning service. With SageMaker, data scientists and developers can quickly and easily build and train machine learning models, and then directly deploy them into a production-ready hosted environment.
- COMPLEX PRICING
	- Can create lots of resources


---


### <span style="color: #C0392B;">Hard Limits & Pivots</span>

- **DynamoDB Item Size:**     400KB     =>     **S3 Pointer Pattern**
- **S3 Single PUT:**          5GB       =>     **Multipart Upload**
- **RDS Read Replicas:**      5         =>     **Aurora** (max 15)
- **Lambda Execution Time:**  15 mins   =>     **Fargate / Batch / EC2**
- **Lambda Package Size:**    250MB     =>     **Container Images**
- **API GW Timeout:**         29s       =>     **Async Pattern** (SQS)
- **VPN Tunnel Bandwidth:**   1.25 Gbps =>     **TGW + ECMP**
- **TGW Bandwidth Limit:**    50 Gbps   =>     per VPC attachment
- **Migration Scale:**        Petabytes = **Snowball** => Exabytes = **Snowmobile**.

---

### <span style="color: #D35400;">Exam Triggers</span>

- **DB in 3 AZs:** ==Minimum 6 subnets==.
- **AZ independence (3 AZs):** ==3 NAT Gateways + 4 Route Tables==.
- **Encrypt data over DX:** ==Public VIF + IPsec VPN==.
- **Inspect packet payload:** ==Traffic Mirroring== or **GWLB**.
- **Audit "Who did what":** ==CloudTrail==.
- **Trace microservice latency:** ==X-Ray==.
- **Query multi-account inventory:** ==Resource Data Sync + Athena==.

---

### <span style="color: #16A085;">Messaging & Events</span>

- **SQS DLQ:** Quarantines "poison pill" messages to prevent infinite recycling.
- **DynamoDB Streams:** Tracks item-level changes to pass events to Lambdas.

---
---

## OLD MEMOS UNDER HERE

- What is SCP?

> Can Resource Access Manager share 'VPCs' between AWS accounts?
> A: No - but can share subnets

- Resource Access Manager
- VPC sharing between accounts
^ Understand VPC sharing and why subnets can be shared.

> What AWS feature allows you to implement admin permissions delegation within AWS?
- Policy boundaries

> A bucket in account B allows access to IAM users in account A. Who will own objects being added into bucket B?
- Who will own objects in bucket

> Users from account A access a bucket in account B by running sts:AssumeRole on an IAM role in account B. Who will own objects added to a bucket in account B?
- Why?? ^
- sts:AssumeRole

- cluster placement group
- enhanced networking

- IPSEC VPN
- TGW and CGW
- Gateway endpoint into VPC

- EMR cluster

- Session stickiness on target group
- What is target group?

CLOUDFRONT!!
- Origin Protocol Policy
- Trusted signers
- field level encryption
- What is an origin fetch?
- What steps do you need to do to add an SSL certificate to a CloudFront distribution?
![[Pasted image 20260508080044.png|131]]

[^1]: An _instance store_ provides temporary block-level storage for your instance. This storage is located on disks that are physically attached to the host computer. Instance store is ideal for temporary storage of information that changes frequently, such as buffers, caches, scratch data, and other temporary content, or for data that is replicated across a fleet of instances, such as a load-balanced pool of web servers.
