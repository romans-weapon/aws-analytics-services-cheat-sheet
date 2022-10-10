# AWS Data Analytics Services Cheat Sheet

A last minute revison of some important data analytics services in AWS for quick preparation. As a part of this doc we will have info about the below mentioned services:

### Table of Contents
- [Kinesis Data Streams/ Data FireHose and Data Analytics](#Kinesis) 
- [S3](#S3)
- [DynamoDB](#DynamoDB)
- [Lambda](#Lambda)
- [Glue (Data Catalogue and ETL)](#Glue)
- [EMR](#EMR)
- [OpenSearch](#OpenSearch)
- [RedShift](#RedShift)
- [Athena](#Athena)
- [Quicksight](#Quicksight)


## Kinesis

### Kinesis Data Streams
- Kinesis Data Streams is a service which allows you to stream BigData into your systems.
- A stream in Kinesis is made of shards. A stream can have any number of shards and must be provisioned during creation of the stream.
- Number of shards in the stream determine the stream capacity in terms of ingestion and consumption. Data ingested is split across these shards.
- Producers producing to the stream can produce data at the rate of 1MB/sec or 1000 records/sec per shard.(Ex: A stream having 6 shards has 6Mb/sec ingest capacity)
- Consumers reading the shards of the stream can consume  data at the rate of 2MB/sec or 2000records/sec per shard for all consumers and 2MB/sec per shard per consumer if we enable enhanced fan-out option.Each shard can support up to five read transactions per second (200ms latency).For Enhanced fan-out we have (~70ms latency).
- There are two types of capacity modes 1.Provisioned Mode -if we can plan the capacity in advance and 2. On-Demand Mode - if we dont know the capacity of stream in advance(provides 4MB/sec capacity and cales on demand)
- Kinesis Data Streams are deployed within a region and can be accessed using IAM policies or interface VPC Endpoint.
- Has Encryption in flight (HTTPS) and encryption at rest (KMS).
- Kinesis integrates with the following:
   - Producers: SDK/KPL/Kinesis Agent/Spark/Kafka/Nifi
   - Consumers: Kinesis Data Analytics/Spark/Firehose/Lambda/KCL/SDK
- Kinesis Producer Library (KPL) -A java Library with the auto retry/asynchronous producing of data/Batching mechanism.
- When Not to Use the KPL :
       -  The KPL can incur an additional processing delay of up to RecordMaxBufferedTime within the library (user-configurable). Larger values of RecordMaxBufferedTime results in higher packing efficiencies and better performance. Applications that cannot tolerate this additional delay may need to use the AWS SDK directly.
- Kinesis Client Library (KCL) - Read records from stream produced by KPL .Has checkpointing feature to resume progress and leverages DynamoDb for checkpointing (Expired Iterator exception - increase WCU of DynamoDbTable).
- Shards in a stream can be splitted or merged called as Re-sharding.Shard splitting can be used to increase stream capacity or to divide a hot shard.It takes 30k secs for doubling shards from 1000 to 2000.We can increase the number of shards of the Kinesis stream by using the UpdateShardCount API.
- Autoscaling is not a native feature in Kinesis Data Streams. Re-sharding shards(split or merge) can cause out of order processing of records and also can induce duplication at the consumer side.In order to resolve out-of order processing we must read the parent shard completely before reading the child shard.KCL has the logic built in to address this issue.
  For duplication we must implement a logic by adding unique id for each record getting consumed and processed.

#### Kinesis vs Kafka vs SQS 

| Kinesis Data Stream | Apache Kafka | SQS Queue
| ------ | ------ |------------| 
| Contains Steams | Contains topics | Contains Queues |
| Streams contain multiple shards | Topics contain multiple partitions |Queues contain message Group Id |
| Can do shard splitting and merging | Can only add topics to partition | No Limit
| 1MB message size hard limit | 1MB message size ,can configure for higher  | 256KB (more if extended library is used)
| Unique id for each record is Seq. number |Unique id for each record is Offset number| N/A
| Kinesis writes data to 3 machines synchronously | Replication factor can be configured| N/A
| Data Retention from 1day (default) to 365 days | 7 days by default and can be configured |4 to 14 days retention


### Kinesis Data Firehose
- Kinesis Data Firehose is a fully managed service which is used to send data in near-real time to target destination after performing some transformations.
- Kinesis Firehose integrates with the following:
   - Producers: Applications/ SDK /KPL/Kinesis Agent/Kinesis Data Streams/CW/IOT
   - Consumers: S3/Redshift (using COPY command from S3)/ ES/DataDog/Splunk /HTTP Endpoint
- It is a near-real time service (60s min. latency) and has auto-scaling capacity.
- Data transformations are done using Lambda and it supports may data formats.
- Spark streaming and KCL donot read from KDF.
- Firehose accumulates data inside a buffer.The buffer limits are defined by buffer time and buffer size. Min buffer time is 1 min and buffer sie is Few Mb.There is no data storage in Firehose.
- With KPL we can produce to Streams and Firehose.
- Uses IAM role for accessing other services from Firehose and it also has in flight and at rest encryption.

### Kinesis Data Analytics
- A serverless service used to query ,transform and analyze streaming data in real time with Apache Flink.Kinesis analytics reads data from a stream and writes to other input streams.
- Kinesis Data Analytics support two types of inputs:
   - A streaming data source is continuously generated data that is read into your application for processing.
   - A reference data source is static data that your application uses to enrich data coming in from streaming sources.
- Using Kinesis Data Analytics you can quickly build SQL queries and sophisticated Apache Flink applications in a supported language such as Java or Scala using built-in templates and operators for common processing functions to organize, transform, aggregate, and analyze data at any scale.
- SQL queries in your application code execute continuously over in-application streams. An in-application stream represents unbounded data that flows continuously through your application. Therefore, to get result sets from this continuously updating input, you often bound queries using a window defined in terms of time or rows. These are also called windowed SQL.
- Kinesis Data Analytics supports the following window types:
     - Staggered Window: A query used to aggregate and analyze groups of data that arrive at inconsistent times. 
     - Tumbling window: A query that aggregates data using distinct time-based windows that open and close at regular intervals.
     - Sliding Windows: A query that aggregates data continuously, using a fixed time or rowcount interval.
- Kinesis Analytics integrates with the following:
   - Producers: Kinesis Data Streams/ Kinesis Data Firehose /S3 for Reference tables
   - Consumers:  Kinesis Data Streams/ Kinesis Data Firehose /Lambda
- Kinesis Data Analytics provisions capacity in the form of Kinesis Processing Units (KPU). A single KPU provides you with the memory (4 GB + 1 vCPU) and corresponding computing and networking.

## S3 (Simple Storage Service)
- S3 is a first AWS cloud service from Amazon.It sis used to store any kind of data(i.e.., servers as a data lake).
- S3 stores data as objects(files) within buckets(buckets).It is a global service.
- Each object in S3 has a key. A key is the full path of the object Ex:For an object stored in  <bucket_name>/folder1/fol2/file.txt - **/folder1/fol2/file.txt**  acts as a key , **/folder1/fol2/** acts as prefix and  **file.txt** is the actual object.
- Max size of an object is 5TBand there is no limit for the number of objects stored(infinitely scalable). Used multipart upload for object size >100MB and should use multipart upload if size > 5GB.Multipart upload loads the data parallely into S3.
- S3 has different storage classes:
     - S3 Standard -For Big Data Analytics/Mobile apps etc.Used for frequently accessed data.Has less latency and high throughput.
     - S3 Standard-IA - For data that is less frequently accessed(not accessed for 30 days) and hs rapid access when needed. Less cost than standard type.
     - S3 Standard-One ZoneIA - For data that is less frequently accessed(not accessed for 30 days) and hs rapid access when needed. Data stored in single AZ.
     - S3 Glacier - 
         - Glacier Instance retrieval - Great for data access once a quarter (90 days).Millisec data retrival
         - Glacier Flexible retrieval- Great for data access once a quarter (90 days). Can wait upto 12hrs for data retrieval
         - Glacier Deep Archive - Great for data access once in 180 days. Can wait upto 48 hours for retrieval.Lowest cost.
    - S3 Intelligent Tiering - Automatic movement of objects between storage classes 
- S3 LifeCycle Rules:
   - Transition Actions - Defines when objects are transitioned to other storage classes.
   - Expiration Actions - Define when objects will be expired(deleted) from the bucket
- S3 versioning can be done if enabled. Once enabled for every new upload of an existing file a new version ID will be created. Any file not versioned pior to enabling versioning will be set to null.
- S3 Replication -must enable versioning in source and dest for doing replication.There is no concept of replication chaining.
   - Cross Region Replication - Async replication to a bucket in different region.Used for compliance/lower-latency access
   - Same Region Replication - Log Aggr. live replication between diff accounts.
- S3 automatically scales to high request rates and gives 5500 GET and 3500 PUT/POST per prefix.
- S3 Transfer Acceleration is used to increase transfer speed by transferring a file to edge location which will then forward the data to S3 bucket in target region.
- S3 Encryption:
   - Server-side Encryption using
       - Amazon S3-Managed Keys (SSE-S3) -AWS managed encryption Keys    
       - AWS KMS-Managed Keys (SSE-KMS) - AWS Key Management service to manage keys
       - Customer-Provided Keys (SSE-C) - Client provided keys. Mandatory to used https 
   - Client-side Encryption using
      - Client-side master key - Encryption is done at client side
- S3 Security 
  - User based - IAM policies
  - Resource based - Bucket policies /Object ACL's
  - Network based - support Gateway VPC endpoint for other services to access S3 via private network.
- S3 Select and Glacier Select are used to query s3 data in place. Can do select on uncompressed csv files.

#### Storage services comparision: S3 vs EBS vs EFS vs FSX
| S3 | EBS | EFS | FSx |
| ------ | ------ |------ | ----- |
| S3 is an object storage service. | EBS is a block storage service |EFS is a file storage system.| Fsx is a storage system for windows
| Designed to provide archiving and data control options and to interface with other services beyond EC2  | Designed to act as storage for a single EC2 instance| Designed to provide scalable storage for multiple EC2 instances | Designed as a storege for Windows
| Web Interface | File System interface | Web and file system interface | File system interface for windows.
| Infinitely Scalable | Hardly Scalable | Scalable |  Fully scalable 
| Slower than EBS and EFS | Faster than S3 and EFS | Faster than S3, slower than EBS  | Faster
| Good for storing backups and other static data | Is meant to be EC2 drive.Only available with EC2 | Good for applications and shareable workloads| Good for comput internsive workloads
|Used mostly in Big Data Analytics,Backups/archives/ Web content management|Boot volumes,Transaction and No-SQL Databases|Big Data Analytics,Backups/archives| Data analytics, media and entertainment workflows, web serving and content management

## DynamoDB
- A No-SQL serverless database on AWS cloud.
- In traditional relational databases there is more of vertical scaling and very little horizontal scaling(in trems of increasing read replicas which is having limits)
- No-SQL databases are non-relational databases which are distributed which give horizontal scalability. They dont support query joins/aggregations i.e.., all the data is present in one single row.
- DynamoDB is a fully managed,highly available with replication across multiple Az's HAVING 100s of TB of storage.
- It is low cost and has auto-scaling capability.
- DynamoDB is made up of tables. Each table has partition key or a combination of partition key + sort key which is together called primary key.Each table can have infinite number of rows.Each row/item can have max. size of 400KB 
- Use cases - Mobile Gaming/Metastore for amazon s3 objects/live voting/ec-commerce cart/OLTP use cases/web-session management.
- DynamoDB -Read/Write Capacity modes:
    - Provisioned mode - if you can predict the capacity beforehand .Must provision Read and Write Capacity units
    - OnDemand Mode - if you don't know the capacity prior to creation of the table.More expensive than provisioned.
- We must provision 1. RCU Read Capacity Units -throughput for reads 2. WCU Write capacity Units - throughput for writes
  - WCU - 1 WCU Represents one Write per sec for item/row of size 1kb.For items > 1KB more WCU are consumed.
  - RCU - 1 RCU represents one strongly consistent read and 2 eventually consistent reads per second for an item of size 4KB.
- Data in DynamoDb is stored in partitions.WCU's and RCU's are spread evenly across all partitions.
- When we exceed the provisioned capacity of the table we get a provisioned throughput exception. This is beacuse of 1. We have a hot partition 2. Hot Key 3. very large items.Can be resolved by 1.Exponential backoff 2. Distribute the partition key  or 3. If it is RCU issue we enable DAX.
- Local Secondary Index (LSI) - An alternative sort key for your table .Must be created at the time of table creation and cannot be changed later.We can have upto 5 LSI per table. Attribut projections may contain some or all attributes of the base table.Uses WCU nad RCU of main table
- Global Secondary Index(GSI)- An alternative primary key 9partition + Sort key) for teh table aprt from that of main table.Used to speed up queries on non-key attributes.Must have WCU and RCU defined for the GSI. If the provisioned WCU and RCU throttle on the GSI then the main table will also be throttled.Can have 20 GSI per table.
- DynamoDB Accelerator(DAX) is a fully manages in-memory cache for DynamoDB with microsecond latency.We need to create a DAX cluster and it acts as a layer between the app and the DyanamoDB  table to get very fast reads.It solves the Hot key problem.
- DynamoDB Streams - Ordered streams of item level modifications in a DynamoDB table made of shards .These streams can be sent to Kinesis Data Streams/ Lambda/KCL etc. Data retention is 24 hrs.
- DynamoDB TTL- Expire items after a specific timestamp.Mostly used in web session management.
- DynamoDB Queries  finds items based on primary key values. You can query any table or secondary index(GSI or LSI) that has a composite primary key (a partition key and a sort key).You must specify the partition key name and value as an equality conditiona and optionally provide a second condition for the sort key. The sort key condition must use one of the following comparison operators: =, <, <=, >, >=, BETWEEN, AND.
- DynamoDB Scan operation reads every item in a table or a secondary index. By default, a Scan operation returns all of the data attributes for every item in the table or index.An inefficient way of reading data as it consumes lot of RCU's.
- DynamoDB can be accessed using gateway VPC endpoint from other services without the internet.Access to tables is also fully controlled by IAM.Has in-filght and at rest encrypton.
- DynamoDB integrations: 
     - Producers: Client SDK,DMS (Database Migration Service),AWS Data Pipeline
     - Consumers: DynamoDB Streams, Glue(metadata store), EMR (hive)
     
## Glue
- Glue is a serverless service which does data discovery and also for ETL(Extract/Transform/Load) of the Data for Analytics.
- Glue consists of two major components
    - Glue Data Catalog - Central metadata Repository for your Data Lake
    - Glue ETL - ETL pipelines which are trigger driven/on a particular schedule/or on demand
- Glue crawlers scans the data in S3 and creates a schema on top of it and store the metadata in Glue Data Catalog.
- The crawler contains a classifier which reads the data in a data store. If it recognizes the format of the data, it generates a schema. The classifier also returns a certainty number to indicate how certain the format recognition was.
- Glue provides a set of built-in classifiers, but you can also create custom classifiers. AWS Glue invokes custom classifiers first, in the order that you specify in your crawler definition. Depending on the results that are returned from custom classifiers, AWS Glue might also invoke built-in classifiers. If a classifier returns certainty=1.0 during processing, it indicates that it's 100 percent certain that it can create the correct schema. AWS Glue then uses the output of that classifier.
- If no classifier returns certainty=1.0, AWS Glue uses the output of the classifier that has the highest certainty. If no classifier returns a certainty greater than 0.0, AWS Glue returns the default classification string of UNKNOWN.
- Once metadata is Cataloged it can be used to treat your unstructured data as structured.
- Glue can be used as a metastore for Hive and also Hive metastore can be converted to glue catalog table.
- Glue ETL will automatically generate code in spark or scala for you for transforming/clean/enrich data and runs using spark under the hood which can be modified by us or we can provide our own python scripts.
- Glue's extension of the PySpark Scala dialect for scripting extract, transform, and load (ETL) jobs.
- Can provision additional DPU's to increase the performance of underlying spark jobs. We can enable job metrics to study and set the max. DPU's for your job.
- Target can be S3/JDBC(RDS/Redshift)/Data Catalog.Glue Triggers automate running of jobs. The transformations in Glue ETL are applied on DynamicFrame similar to DataFrame.
- Development Endpoints allow developing ETL scripts using a notebook and later run it through ETL job.These Endpoints in VPC controlled by security groups.
- Glue jobs are run using schedule/job bookmark(to persist state to prevent reprocessing the data)/cw events.
- Glue Integrations:
    - Sources : S3/DynamoDB/JDBC(on-prem or RDS)
    - Target : Redshift/EMR/Athena/CW for monitoring

## EMR -Elastic Map Reduce
- EMR is a managed hadoop framework on AWS. It is Hadoop installed on a fleet of EC2 instances which allowa us to run Big Data workloads with vast amount of data
- EMR supports powerful and proven Hadoop tools such as Hive, Pig, HBase, and Impala. Additionally, it can run distributed computing frameworks besides Hadoop MapReduce such as Spark or Presto using bootstrap actions. 
- EMR Components:
  - Clusters:  A collection of EC2 instances. You can create two types of clusters
        - a **transient cluster** that auto-terminates after steps complete.
        - a **long-running cluster** that continues to run until you terminate it deliberately.
  - Nodes: Each EC2 instance is called as a Node. EMR has 3 types of Nodes in its architecture:
       - Master Node : A node that manages the cluster by running software components to coordinate the distribution of data and tasks among other nodes for processing.It tracks the status of tasks and monitors the health of the cluster.
       - Core Node : A node which stores data in HDFS and runs tasks in the EMR cluster.
       - Task Node : A node which doesnt store any data and is only used for running tasks.Tasks nodes are optional.
- EMR Architecture
    - Storage – this layer includes the different file systems that are used with your cluster.
        - Hadoop Distributed File System (HDFS) – a distributed, scalable file system for Hadoop.
            - HDFS distributes the data it stores across instances in the cluster, storing multiple copies of data on different instances to ensure that no data is lost if an individual instance fails.
            - HDFS is ephemeral storage.
            - HDFS is useful for caching intermediate results during MapReduce processing or for workloads that have significant random I/O.
        - EMR File System (EMRFS) – With EMRFS, EMR extends Hadoop to directly be able to access data stored in S3 as if it   were a file system.The EMR File System (EMRFS) is an implementation of HDFS that all EMR clusters use for reading  and writing regular files from EMR directly to S3. It provides the convenience of storing persistent data in S3 for  use with Hadoop while also providing features like consistent view and data encryption.
        - Local File System – refers to a locally connected disk.
    - Cluster Resource Management – this layer is responsible for managing cluster resources and scheduling the jobs for processing data. By default, Amazon EMR uses YARN, which is a component introduced in Apache Hadoop 2.0 to centrally manage cluster resources for multiple data-processing frameworks.
- Scaling
     - There are two main options for adding or removing capacity:
          - Deploy multiple clusters: If you need more capacity, you can easily launch a new cluster and terminate it when you no longer need it. There is no limit to how many clusters you can have.
          - Resize a running cluster: You may want to scale out a cluster to temporarily add more processing power to the cluster, or scale in your cluster to save on costs when you have idle capacity. When adding instances to your cluster, EMR can now start utilizing provisioned capacity as soon it becomes available. When scaling in, EMR will proactively choose idle nodes to reduce impact on running jobs.
- Security
    - EMR integrates with IAM to manage permissions. You define permissions using IAM policies, which you attach to IAM users or IAM groups. The permissions that you define in the policy determine the actions that those users or members of the group can perform and the resources that they can access.
    - EMR uses IAM roles for the EMR service itself and the EC2 instance profile for the instances. These roles grant permissions for the service and instances to access other AWS services on your behalf. There is a default role for the EMR service and a default role for the EC2 instance profile.
    - EMR uses security groups to control inbound and outbound traffic to your EC2 instances. When you launch your cluster, EMR uses a security group for your master instance and a security group to be shared by your core/task instances.
    -Encrypting data in transit using Transport Layer Security (TLS) (as described in the EMR documentation). You can do either of the following:
         - Manually create PEM certificates, zip them in a file, and reference from Amazon S3.
         - Implement a certificate custom provider in Java and specify the S3 path to the JAR.
    - EMR supports optional S3 server-side and client-side encryption with EMRFS to help protect the data that you store in S3. It also supports launching clusters in a VPC.
    - EMR security configurations use a combination of open-source HDFS encryption and LUKS encryption (Local disk encryption for root EBS volumes).
    - Block Public Access configuration is an account-level configuration that helps you centrally manage public network access to EMR clusters in a region. You can enable this configuration in a region and block your account users from launching EMR clusters that allow unrestricted inbound traffic from the public IP address (source set to 0.0.0.0/0 for IPv4 and ::/0 for IPv6) through its ports.
- S3DistCp command is the right thing to do to copy data from S3 into HDFS and then make sure the data is processed locally by the EMR cluster MapReduce job.Upon completion, you will use S3DistCp again to push back the final result data to S3.
- You can create Amazon EMR clusters with custom Amazon Linux AMIs from the Amazon EMR console, AWS Command Line Interface (CLI), or the AWS SDK with the Amazon EMR API. Custom AMIs are supported on Amazon EMR release 5.7.0 or later.  
- Amazon EMR offers two ways to scale your clusters: you can either use Amazon EMR's support for Auto Scaling , or EMR Managed Scaling.
    - If you’re running Apache Spark, Apache Hive, or YARN-based applications and want a completely managed experience, you can use EMR Managed Scaling. If you need to define custom rules involving custom metrics for applications running in the cluster, you should use Auto Scaling.
    - When you create a cluster and specify the configuration of the master node, core nodes, and task nodes, you have two configuration options. You can use:
          - Instance fleets
          - Instance groups (provides autoscaling)
- EMR Integrations:
      - S3 using EMRFS
      - Dynamodb using hive to scan the table and use it for processing
      - Glue Data catalogue for metadata of the tables
- Apache DistCp is an open-source tool that you can use to copy large amounts of data. S3DistCp is an extension of DistCp that is optimized to work with AWS, particularly Amazon S3. The command for S3DistCp in Amazon EMR version 4.0 and later is s3-dist-cp, which you add as a step in a cluster or at the command line. 
  Using S3DistCp, you can efficiently copy large amounts of data from Amazon S3 into HDFS where it can be processed by subsequent steps in your Amazon EMR cluster.

## OpenSearch
- A petabyte scale analysis and reporting service in AWS.Amazon OpenSearch lets you search, analyze, and visualize your data in real-time.It is a cobination of Elastic search and Kibana with integrations to LogStash
- Opensearch has 2 major components
      - Documents - Documents are the things we search for in ES. It can be text/JSON.Every Document has a unique id.
      - Index - An index powers search to all documents and finds a result much faster. Each index is split ino shards.Each document is hashed and stored in shards.Each index has 2 primary and 2 replica shards.
- Domains are clusters with the settings, instance types, instance counts, and storage resources that you specify for spinning up a cluster.
- We can create multiple Elasticsearch indices within the same domain. Elasticsearch automatically distributes the indices and any associated replicas between the instances allocated to the domain.
- ElasticSearch storage used EBS volumes for storing hot data for fast performance.Used S3+caching for warm storage and S3 for cold storage.
- Launching three dedicated master nodes is the best and avoids split brain problem.
- You can load streaming data from the following sources using AWS Lambda event handlers:
    - Amazon S3
    - Amazon Kinesis Data Streams and Data Firehose
    - Amazon DynamoDB
    - Amazon CloudWatch
    - AWS IoT
- Memory Pressure in JVM in OS can result if we have too many shards in the cluster or have unbalanced shard allocations.This can be resolved by lowering the shards in the cluster.
- OpenSearch Integrations:
      - Kinesis Data Firehose
      - IOT and CW

## Athena
- A serverless querying tool used to query data in S3 using SQL without loading data into it.
- Uses Presto, an open source, distributed SQL query engine optimized for low latency, ad hoc analysis of data.
- Athena supports a wide variety of data formats such as CSV, JSON, ORC, Avro, or Parquet.
- Athena automatically executes queries in parallel, so that you get query results in seconds, even on large datasets.
- Athena uses Amazon S3 as its underlying data store to store query results, making your data highly available and durable.
- Athena integrates with Amazon QuickSight for easy data visualization.
- Athena uses Glue Data Catalog as metadata store to impart structure to the data in S3.
- Athena uses workgroups to organize users/teams/applications etc. to control query access and track cost by work group and set up 
  alarms to notify using SNS topics.Can Limit data per query or per work-group.
- We can save a lot of cost by using columnar formats and also by partitioning your data so that less data is scanned. Only successful or cancelled queries count,failed queries are not charged.
- Has encryption at rest in S3 and in flight encryption between S3 and Athena.
- Better performance can be obtained by using columnar data or by merging small files into larger files to reduce I/O.
- Athena supports ACID transactions by using table_type='ICEBERG'.Can improve performance by using OPTIMIZE command to compact small files.
- Athena, recommendS using either Apache Parquet or Apache ORC file formats, which compress data by default and are splittable. When they are not an option, then try BZip2 or Gzip with an optimal file size.
- Athena query result location in S3 supports CSE-KMS, SSE-KMS or SSE-S3 only not customer provided encryption keys(SSE-C)
- Athena Limitations:
     - Stored procedures are not supported.
     - Athena does not support querying the data in the S3 Glacier flexible retrieval or S3 Glacier Deep Archive storage classes.
     - MERGE AND UPDATE Statements are not supported unless you use table_type='ICEBERG'
- Athena Integrations:
      - sources: Glue Data Catalog and S3
      - Targets: Quicksight/JDBC/ODBC tools/Zeppelin notebooks 

## Redshift
- A petabyte-scale data warehouse service on AWS extends data warehouse queries to your data lake which allows you to run analytic queries against petabytes of data stored locally in Redshift, and directly against exabytes of data stored in S3.
- Redshift uses columnar storage, data compression for storing the data.Uses MPP (massively parallel processing) data warehouse architecture to parallelize and distribute SQL operations.
- Redshift automatically and continuously backs up your data to S3. It can asynchronously replicate your snapshots to S3 in another region for disaster recovery.
- Redshift currently only supports Single-AZ deployments.
- Redshift Nodes
    - The leader node receives queries from client applications, parses the queries, and develops query execution plans. It then coordinates the parallel execution of these plans with the compute nodes and aggregates the intermediate results from these nodes. Finally, it returns the results back to the client applications.
    - Compute nodes execute the query execution plans and transmit data among themselves to serve these queries. The intermediate results are sent to the leader node for aggregation before being sent back to the client applications.
    - Node Type
         - Dense storage (DS) node type – for large data workloads and use hard disk drive (HDD) storage.
         - Dense compute (DC) node types – optimized for performance-intensive workloads. Uses SSD storage.
         - RA3( Managed storage) - you can choose the number of nodes based on your performance requirements, and only pay for the managed storage that you use.
- Parameter Groups – a group of parameters that apply to all of the databases that you create in the cluster. The default parameter group has preset values for each of its parameters, and it cannot be modified.
- By using Enhanced VPC Routing, you can use VPC features to manage the flow of data between your cluster and other resources.
- Redshift spectrum is a serverless scalable layer which enables you to run exabytes of data in S3 without having to load or transform any data.It comes along with redshift we need to configure it separately.
- To use redshift spectrum,you need to do the following:
    - Create an IAM role for Amazon Redshift 
    - Associate the IAM role with your cluster
    - Create an external schema and an external table 
        - Ex: create external schema myspectrum_schema from data catalog database 'myspectrum_db' iam_role 'arn:aws:iam::123456789012:role/myspectrum_role'  create external database if not exists; create external table (sales int,listid int) stored as textfile  location 's3://redshift-downloads/tickit/spectrum/sales/';
    - Query your data in Amazon S3
- Redshift Data Share is a secure way to share live data across Redshift clusters within an AWS account, without the need to copy or move data.Ex: to move data from development to production envs.


## QuickSight
- Amazon QuickSight is a fast business analytics service to build visualizations, perform ad hoc analysis, and quickly get business insights from your data. It is an analytics service that you can use to create datasets, perform one-time analyses, and build visualizations and dashboards. I
- Amazon QuickSight seamlessly discovers AWS data sources, enables organizations to scale to hundreds of thousands of users, and delivers fast and responsive query performance by using a robust in-memory engine (SPICE).
- You can refresh your SPICE datasets at any time. Refreshing imports the data into SPICE again, so the data includes any changes since the last import. You can refresh SPICE data by using any of the following approaches:
    - You can use the options on the 'Your Data Sets' page.
    - You can refresh a dataset while editing a dataset.
    - You can schedule refreshes in the dataset settings.
    - You can use the CreateIngestion API operation to refresh the data.
- Bar charts -for comparision/Distribution
- Line Chart - trends over time
- Scatter plots/HeatMaps -Correlation. Highlight outliers and trends using color.
- Pie Chart - Aggregation
- Donut Chart - Percentage of total amt
- Gague chart - compare values in a a measure
- Pivot table - Tabular data
- KPi's - Compare key value to its target value
