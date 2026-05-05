---
title: "AWS purpose-built database recovery: A guide to business continuity and disaster recovery strategies"
url: "https://aws.amazon.com/blogs/database/aws-purpose-built-database-recovery-a-guide-to-business-continuity-and-disaster-recovery-strategies/"
date: "Wed, 22 Apr 2026 15:38:17 +0000"
author: "Marco Tamassia"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
<p>Modern applications often use multiple AWS purpose-built databases to handle different workloads efficiently. When disruptions occur, maintaining data consistency and alignment across these databases becomes complex.</p> 
<p>This post addresses recovery challenges in multi-database architectures, focusing on both low-consistency and mission-critical scenarios. We explore practical strategies for implementing resilient recovery mechanisms across <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a>, <a href="https://aws.amazon.com/rds/aurora/" rel="noopener noreferrer" target="_blank">Amazon Aurora</a>, <a href="https://aws.amazon.com/neptune/" rel="noopener noreferrer" target="_blank">Amazon Neptune</a>, <a href="https://aws.amazon.com/opensearch-service/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service</a>, and other AWS database services.</p> 
<h2>The purpose-built database approach</h2> 
<p>At the core of AWS database offerings lies a comprehensive suite of <a href="https://aws.amazon.com/products/databases/" rel="noopener noreferrer" target="_blank">purpose-built database services</a>, each designed to meet the unique requirements of modern applications. These purpose-built databases provide a high-performance, secure, and reliable foundation to power a wide range of workloads, from <a href="https://aws.amazon.com/ai/generative-ai/" rel="noopener noreferrer" target="_blank">generative AI solutions</a> to data-driven applications that drive value for businesses and their customers.</p> 
<p>While a one-size-fits-all database approach might suffice for straightforward scenarios, modern microservice architectures benefit significantly from using <strong>purpose-built databases</strong>. You can use this approach to select the optimal database and data model for each service based on its specific requirements – whether that’s transaction consistency, query patterns, scalability needs, or security constraints. Each database can be independently improved and scaled according to its workload characteristics.</p> 
<p>However, this architectural flexibility introduces complexity when disruptions occur. The challenge lies in coordinating recovery across multiple databases while maintaining data consistency and alignment. This is where AWS database services’ built-in high availability and data protection features become critical.</p> 
<p>Features like automated backups, point-in-time recovery, cross-Region replication, and automated failover provide the foundation for building resilient multi-database architectures. Understanding how to use and coordinate these capabilities across your purpose-built databases is essential for effective business continuity and disaster recovery planning.</p> 
<h2>Different levels of requirements</h2> 
<p>Not all purpose-built database architectures are created equal. The requirements for data consistency and recovery strategies can vary significantly, depending on the nature and criticality of the application. In this post, we explore two distinct scenarios that illustrate the different levels of requirements that organizations might face when aligning their purpose-built databases:</p> 
<ul> 
 <li><strong>Use case I: </strong>Applications with no consistency requirements – In this scenario, the applications have no strict requirements in terms of data consistency among the multiple backend databases. These could be low-traffic, non-critical applications where the databases operate in relative isolation, and their synchronization isn’t a significant concern. For such use cases, the focus might be more on maintaining individual database reliability rather than maintaining strict data alignment across the entire ecosystem. The recovery strategies in this case would prioritize simplicity and operational efficiency over complex data reconciliation processes.</li> 
 <li><strong>Use case II</strong>: Applications with high consistency requirements – This category includes mission-critical applications that demand strict data consistency and integrity across the purpose-built databases. These could be applications that handle sensitive financial data, healthcare records, or other mission-critical workloads where data alignment is paramount. For such use cases, the recovery strategies must prioritize data consistency and provide robust mechanisms for seamless data reconciliation, even in the face of complex disruptions or failures.</li> 
</ul> 
<p>In this post, we provide you with the necessary insights and best practices to recover and align your purpose-built databases and navigate through potential recovery scenarios, regardless of your specific application requirements.</p> 
<h2>Use case 1: Applications with no requirements in terms of data consistency among the multiple backends</h2> 
<p>In this scenario, we consider applications where strict data consistency across multiple purpose-built databases isn’t a critical requirement. These are typically low-traffic, non-mission-critical applications where the databases operate relatively independently, and their synchronization isn’t a significant concern. For such use cases, the focus is more on maintaining individual database reliability and operational efficiency rather than maintaining strict data alignment across the entire ecosystem.</p> 
<p>Let’s explore a use case that illustrates this scenario: a multi-channel customer feedback system. It incorporates different database services such as Amazon Aurora, Amazon DynamoDB, <a href="https://aws.amazon.com/redshift/" rel="noopener noreferrer" target="_blank">Amazon Redshift</a>, and <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3) as object storage. Imagine a large retail company that wants to capture and analyze customer feedback from various channels: in-store surveys, online reviews, social media comments, and customer service interactions. The company aims to gain insights into customer satisfaction and product performance without requiring real-time consistency across all data sources.</p> 
<h3>Architecture overview</h3> 
<p>The following diagram shows the architecture of the first use case.</p> 
<p><img alt="" class="aligncenter wp-image-70091 size-full" height="878" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DB-4252-1.jpg" width="1138" /></p> 
<p>In this architecture, each database serves a specific purpose, designed for its particular workload. The lack of strict consistency requirements allows for a more flexible and scalable system.</p> 
<p>The solution consists of the following key components and relationships:</p> 
<ul> 
 <li><strong>Web application</strong> – Uses <a href="https://aws.amazon.com/elasticbeanstalk/" rel="noopener" target="_blank">AWS Elastic Beanstalk</a> and serves as the frontend for collecting customer feedback from various channels. It manages the flow of data to appropriate storage services.</li> 
 <li><strong>Aurora</strong> – Stores structured data from in-store surveys and online reviews. This relational database is recommended for handling complex queries and transactions related to customer feedback.</li> 
 <li><strong>DynamoDB</strong> – Captures and stores real-time social media comments and sentiment analysis results. Its low-latency and high-throughput capabilities make it a good fit for handling the constant stream of social media data.</li> 
 <li><strong>Amazon S3</strong> – Acts as a data lake, storing raw data from all sources, including unstructured data like customer service call recordings and chat logs.</li> 
 <li><strong>Amazon Redshift</strong> – Serves as the data warehouse for analytical queries and reporting, combining data from all sources for long-term trend analysis and business intelligence (BI).</li> 
 <li><strong><a href="https://aws.amazon.com/quick/quicksight/" rel="noopener" target="_blank">Amazon Quick Sight</a> – </strong>Connects to Amazon Redshift for creating dashboards, reports, and performing one-time analyses.</li> 
</ul> 
<h3>Data flow</h3> 
<p>The data flow consists of the following steps:</p> 
<ol> 
 <li>Customer feedback is collected through various channels (in-store, online, social media, customer service).</li> 
 <li>The web application routes the data to appropriate storage services: 
  <ol type="a"> 
   <li>Structured survey data goes to Amazon Aurora.</li> 
   <li>Real-time social media data goes to DynamoDB.</li> 
   <li>Raw and unstructured data is stored in Amazon S3.</li> 
  </ol> </li> 
 <li>Data from Aurora and DynamoDB is periodically extracted and loaded into Amazon Redshift, which can be done using <a href="https://aws.amazon.com/what-is/zero-etl/" rel="noopener noreferrer" target="_blank">zero-ETL integration</a>.</li> 
 <li>Amazon S3 data is directly queried by Amazon Redshift using <a href="https://docs.aws.amazon.com/redshift/latest/dg/c-using-spectrum.html" rel="noopener noreferrer" target="_blank">Amazon Redshift Spectrum</a> when needed.</li> 
 <li>BI tools such as Amazon Quick Sight connect to Amazon Redshift for analysis and reporting.</li> 
</ol> 
<h3>Key terms and metrics</h3> 
<p>Before going more into the details of this post, it’s important to provide the definitions of two key aspects when it comes to restore and recovery scenarios:</p> 
<ul> 
 <li><strong>Business continuity</strong> – This refers to the ability of an organization to maintain essential functions during and after a disruption. It encompasses planning and preparation to make sure that an organization can operate its critical business functions during emergency events.</li> 
 <li><strong>Disaster recovery</strong> – This can be considered as a subset of business continuity. It focuses on the IT systems that support business functions, specifically the policies, tools, and procedures to enable the recovery or continuation of vital technology infrastructure and systems following a disaster.</li> 
</ul> 
<p>Two key metrics in business continuity and disaster recovery planning are:</p> 
<ul> 
 <li><strong>Recovery Time Objective (RTO)</strong> – The maximum acceptable length of time that an application can be down after a failure or disaster occurs.</li> 
 <li><strong>Recovery Point Objective (RPO)</strong> – The maximum acceptable amount of data loss measured in time. It’s the age of the files or data in backup storage that must be restored to resume normal operations.</li> 
</ul> 
<p>The following sections describe how these concepts apply to different database services and the options available to meet your business continuity and disaster recovery requirements.</p> 
<h3>Recovery strategies</h3> 
<p>Because the multiple backends have no strict consistency requirements, you can tailor recovery strategies using each database services’ high availability (HA), business continuity, and disaster recovery features. The following table summarizes these options.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td><strong>Architecture Component</strong></td> 
   <td><strong>Feature</strong></td> 
   <td><strong>RTO</strong></td> 
   <td><strong>RPO</strong></td> 
  </tr> 
  <tr> 
   <td rowspan="6"><strong>Amazon Aurora</strong></td> 
   <td><a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html" rel="noopener noreferrer" target="_blank">Built-in high availability and failover</a></td> 
   <td>Can be as low as a few seconds using Aurora’s fast crash recovery and <a href="https://aws.amazon.com/rds/proxy/" rel="noopener noreferrer" target="_blank">Amazon RDS Proxy</a>.</td> 
   <td>Equal to 0, due to the Aurora’s <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.StorageReliability.html#Aurora.Overview.Storage" rel="noopener noreferrer" target="_blank">enhanced durability</a> and <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Reliability.html#Aurora.Overview.AutoRepair" rel="noopener noreferrer" target="_blank">storage auto-repair capabilities of Aurora</a>.</td> 
  </tr> 
  <tr> 
   <td><a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-restore-snapshot.html" rel="noopener noreferrer" target="_blank">Restoring from most recent snapshot</a></td> 
   <td>Recovery time ranges from minutes to hours, depending on database size and configuration. Regular restore testing in production-like environments with representative datasets is recommended to establish accurate recovery time baselines.</td> 
   <td>Can be as low as a few minutes, equal to the age of the most recent snapshot.</td> 
  </tr> 
  <tr> 
   <td><a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-pitr-retained.html" rel="noopener noreferrer" target="_blank">PITR using automated backups</a></td> 
   <td>Minutes to hours, restore time is linear to the database size and the number of incremental changes since the last snapshot.</td> 
   <td>Within LatestRestorableTime and EarliestRestorableTime, depending on the configured backup retention windows. Normally, the LatestRestorableTime is within the last 5 minutes of the current time.</td> 
  </tr> 
  <tr> 
   <td><a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-pitr-bkp.html" rel="noopener noreferrer" target="_blank">PITR using automated backups and AWS Backup</a></td> 
   <td>Minutes to hours, restore time is linear to the database size.</td> 
   <td>Within LatestRestorableTime and EarliestRestorableTime, depending on the configured backup retention windows and backup frequency. Normally the LatestRestorableTime is within the last 5 minutes of the current time.</td> 
  </tr> 
  <tr> 
   <td><a href="https://aws.amazon.com/blogs/database/cost-effective-disaster-recovery-for-amazon-aurora-databases-using-aws-backup/" rel="noopener noreferrer" target="_blank">Using AWS Backup in a cross-region disaster recovery scenario</a></td> 
   <td>Minutes to hours, restore time is linear to the database size.</td> 
   <td>Minutes, depending on the database size and network throughput (the primary and secondary AWS Regions distance does matter).</td> 
  </tr> 
  <tr> 
   <td><a href="https://aws.amazon.com/rds/aurora/global-database/" rel="noopener noreferrer" target="_blank">Amazon Aurora Global Database</a> to improve <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-bcdr-planning" rel="noopener noreferrer" target="_blank">multi-Region resilience</a></td> 
   <td>Minutes</td> 
   <td>Seconds; can be <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-manage-recovery" rel="noopener noreferrer" target="_blank">managed</a></td> 
  </tr> 
  <tr> 
   <td rowspan="3"><strong>Amazon DynamoDB</strong></td> 
   <td><a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/PointInTimeRecovery_Howitworks.html" rel="noopener noreferrer" target="_blank">Enable PITR</a> to protect against accidental writes or deletes and effectively enabling nearly continuous backups of your table data along with enabling deletion protection on the table</td> 
   <td>Service metrics show that 95 percent of customers’ table restores complete in less than one hour. However, restore times are directly related to the configuration of your tables and other related variables. A best practice is to regularly document average restore completion times and establish how these times affect your overall Recovery Time Objective.</td> 
   <td>Up to 5 minutes. PITR continuously backs up your data, and you can restore it to any point in time within the last 35 days, with a potential data loss of up to 5 minutes.</td> 
  </tr> 
  <tr> 
   <td>Use <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/backuprestore_HowItWorks.html" rel="noopener noreferrer" target="_blank">on-demand backups</a> for long-term retention and compliance purposes</td> 
   <td>Service metrics show that 95 percent of customers’ table restores complete in less than one hour. However, restore times are directly related to the configuration of your tables and other related variables. A best practice is to regularly document average restore completion times and establish how these times affect your overall Recovery Time Objective.</td> 
   <td>Determined by backup frequency.</td> 
  </tr> 
  <tr> 
   <td>Implement <a href="https://aws.amazon.com/dynamodb/global-tables/" rel="noopener noreferrer" target="_blank">global tables</a> for multi-Region resilience</td> 
   <td>Zero with <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html" rel="noopener noreferrer" target="_blank">Multi-Region strong consistency (MRSC)</a>.<br /> There’s no need for a restore process because the data is already replicated and available in other replica Regions.</td> 
   <td>Multi-Region Strong Consistency global tables support a Recovery Point Objective (RPO) of zero.</td> 
  </tr> 
  <tr> 
   <td rowspan="3"><strong>Amazon S3</strong></td> 
   <td>Enable <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html" rel="noopener noreferrer" target="_blank">S3 Versioning</a> to protect against accidental deletions or overwrites</td> 
   <td>Near 0, with immediate access to previous object versions.</td> 
   <td>0 because all objects’ versions are retained.</td> 
  </tr> 
  <tr> 
   <td><a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html" rel="noopener noreferrer" target="_blank">Implement S3 cross-Region replication</a> to provide geographical redundancy for disaster recovery</td> 
   <td>Near 0; because the data is already available in the destination Region.</td> 
   <td>Typically, less than 15 minutes, but can vary based on object size and network conditions.</td> 
  </tr> 
  <tr> 
   <td>Use <a href="https://aws.amazon.com/s3/storage-classes/intelligent-tiering/" rel="noopener noreferrer" target="_blank">S3 Intelligent-Tiering</a> for cost-effective long-term storage of infrequently accessed data. Although it’s primarily a cost improvement feature, it can play a role in recovery strategies by making sure that data is available and stored in the most appropriate tier for quick access when needed</td> 
   <td>Milliseconds for data in Frequent Access and Infrequent Access tiers, minutes to hours for data in the Archive Access tier, and within 12 hours for data in the Deep Archive tier.</td> 
   <td>S3 Intelligent-Tiering doesn’t directly affect RPO, but it can be used in conjunction with features like S3 Versioning and S3 cross-Region replication.</td> 
  </tr> 
  <tr> 
   <td rowspan="3"><strong>Amazon Redshift</strong></td> 
   <td>Use automated snapshots (enabled by default to take snapshots of your cluster every 8 hours or 5 GB of data changes) for <a href="https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-snapshots.html#automated-snapshot" rel="noopener noreferrer" target="_blank">PITR</a></td> 
   <td>Typically, 10–45 minutes, depending on the cluster size and data volume.</td> 
   <td>Up to 8 hours or less if 5 GB of data changes occur sooner.</td> 
  </tr> 
  <tr> 
   <td>Implement <a href="https://aws.amazon.com/blogs/aws/automated-cross-region-snapshot-copy-for-amazon-redshift/" rel="noopener noreferrer" target="_blank">automated</a> <a href="https://docs.aws.amazon.com/redshift/latest/mgmt/cross-region-snapshot-copy.html" rel="noopener noreferrer" target="_blank">cross-Region snapshot copy</a> for disaster recovery</td> 
   <td>Depends on cluster size, but typically within 1-2 hours, including snapshot restore time.</td> 
   <td>Depends on the frequency of snapshot creation and copy process.</td> 
  </tr> 
  <tr> 
   <td><a href="https://docs.aws.amazon.com/redshift/latest/dg/c-using-spectrum.html" rel="noopener noreferrer" target="_blank">Redshift Spectrum</a> can be used as part of a recovery strategy to query historical data directly from Amazon S3, reducing the need for frequent data loads</td> 
   <td>Near 0 for querying data, because it doesn’t require data loading.</td> 
   <td>Depends on how frequently data is updated in Amazon S3.</td> 
  </tr> 
 </tbody> 
</table> 
<p><strong>In this use case, each database can be recovered independently without concerns about strict synchronization.</strong> The focus is on making sure that each service maintains its own data integrity and availability. Periodic data reconciliation processes can be implemented to address any potential discrepancies, but they are not critical for the system’s day-to-day operations.</p> 
<p>This approach enables a more flexible, scalable architecture, where each purpose-built database can be improved for its specific workload without maintaining strict consistency across the entire system. It’s particularly suitable for applications where near real-time data alignment is not a requirement, and where the benefits of using specialized databases outweigh the need for immediate cross-database consistency.</p> 
<h2>Use case 2: Applications with high requirements in terms of data consistency among the multiple backends</h2> 
<p>In this scenario, we consider applications that demand strict data consistency and integrity across multiple purpose-built databases. These are typically mission-critical applications that handle sensitive data where maintaining data alignment is paramount. For such use cases, recovery strategies must prioritize data consistency and provide robust mechanisms for seamless data reconciliation, even in the face of complex disruptions or failures.</p> 
<p>An example of a use case that illustrates this scenario is described in <a href="https://aws.amazon.com/blogs/database/building-a-modern-application-with-purpose-built-aws-databases/" rel="noopener noreferrer" target="_blank">Build a Modern Application with Purpose-Built AWS Databases</a>, which describes an online bookstore demo application. This is a modern full-stack web application that serves as an online bookstore with some core features like product catalog management, search functionality, bestsellers tracking, and social recommendations.</p> 
<h3>Architecture overview</h3> 
<p>The following diagram shows the architecture of the second use case.</p> 
<p><img alt="" class="aligncenter wp-image-70093 size-full" height="853" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/09/image-2-3.png" width="1652" /></p> 
<p>We focus on the services that store data, and we describe the techniques that you can use to make this architecture robust and resilient, implementing effective strategies to maintain the consistency of the data.</p> 
<p>The solution consists of the following key components and relationships:</p> 
<ul> 
 <li><strong>Product catalog implemented with Amazon DynamoDB</strong> – Store score product information such as IDs, descriptions and prices. Provides fast key-value lookups and scales from hundreds to billions of products.</li> 
 <li><strong>Enterprise Search system implemented with Amazon OpenSearch Service</strong> – Handles full-text product searches through the main search bar, so users can find books naturally with additional filtering capabilities. Updates flows in real-time through <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html" rel="noopener noreferrer" target="_blank">Amazon DynamoDB Streams</a> and <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a>. A way to improve this pattern is by implementing the <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/OpenSearchIngestionForDynamoDB.html" rel="noopener noreferrer" target="_blank">DynamoDB zero-ETL integration with Amazon OpenSearch Service</a>.</li> 
 <li><strong>Best sellers feature</strong> – Tracks the top 20 purchased books using in-memory sorted sets for microsecond-latency reads. Updates flow in real-time through DynamoDB Streams and Lambda functions.</li> 
 <li><strong>Social Media recommendations implemented with Amazon Neptune</strong> – The graph database is used to store the social connections and track purchase patterns; the goal is to provide traversal of relationships for book recommendations in the online bookstore.</li> 
</ul> 
<p>The following are additional considerations not shown in the diagram:</p> 
<ul> 
 <li><strong>Backup and recovery</strong> – You can use <a href="https://aws.amazon.com/backup/" rel="noopener noreferrer" target="_blank">AWS Backup</a> for consistent, coordinated backups across services (for the complete list of supported services, see the <a href="https://docs.aws.amazon.com/aws-backup/latest/devguide/working-with-supported-services.html" rel="noopener noreferrer" target="_blank">AWS Backup documentation</a>)</li> 
 <li><strong>Monitoring and alerting</strong> – You can use <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> for comprehensive monitoring and alerting.</li> 
 <li><strong>Multi-Region deployment</strong> – You can replicate the entire architecture in a secondary Region for disaster recovery. For detailed guidance on multi-Region architectures, refer to <a href="https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-multi-region-fundamentals/introduction.html" rel="noopener noreferrer" target="_blank">AWS multi-Region fundamentals</a>. This is a 300-level prescriptive guidance designed for cloud architects and senior leaders who build workloads on AWS. This comprehensive guide helps technical decision-makers understand and implement multi-Region architectures to improve workload resilience and high availability.</li> 
</ul> 
<h3>Data flow</h3> 
<p>The data flow consists of the following steps:</p> 
<ol> 
 <li>The web frontend interfaces with the databases systems through API calls and Lambda functions.</li> 
 <li>Changes in the product catalog automatically sync to the search index through DynamoDB Streams and Lambda.</li> 
 <li>Purchase events update both the best sellers list and social recommendations in <a href="https://aws.amazon.com/elasticache/redis/" rel="noopener noreferrer" target="_blank">Amazon ElastiCache for Valkey</a> and Neptune.</li> 
 <li>All systems work together to provide a seamless user experience.</li> 
</ol> 
<h3>Recovery strategies</h3> 
<p>To provide seamless recovery for data alignment across the different purpose-built databases used in this use case, you can implement the following strategies, organized by service:</p> 
<ul> 
 <li><strong>Amazon DynamoDB</strong> – The <a href="https://aws.amazon.com/dynamodb/sla/" rel="noopener noreferrer" target="_blank">DynamoDB availability SLA</a> is 99.99% with standard tables and 99.999% with global tables. However, we still recommend enabling <a href="https://aws.amazon.com/dynamodb/pitr/" rel="noopener noreferrer" target="_blank">point-in-time recovery</a> (PITR), which provides an RPO of zero with multi-Region strong consistency (minutes for single tables), and RTO of minutes or hours, and backup retention of up to 35 days. Optionally, as a secondary protection, <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html" rel="noopener noreferrer" target="_blank">DynamoDB global tables</a> enable efficient cross-Region disaster recovery with strong data consistency. With these approaches, you can set up your DynamoDB tables to be highly resilient in case of any issues, and you can implement <a href="https://aws.amazon.com/blogs/compute/implementing-aws-lambda-error-handling-patterns/" rel="noopener noreferrer" target="_blank">Lambda functions error handling</a> with a <a href="https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html" rel="noopener noreferrer" target="_blank">dead-letter queue (DLQ) mechanism</a>. The recommendation in case of long DynamoDB unavailability is to prevent the access to the frontend, instead of managing a huge number of DLQ entries.</li> 
 <li><strong>Amazon DynamoDB Streams</strong> – A DynamoDB stream is an ordered flow of information about changes to items in a DynamoDB table. When you enable a stream on a table, DynamoDB captures information about every modification to data items in the table. DynamoDB Streams by itself guarantees exactly-once delivery and in-order delivery of item level modifications and stores this information in a log for up to 24 hours. Combining <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html" rel="noopener noreferrer" target="_blank">DynamoDB Time to Live (TTL)</a>, DynamoDB Streams, and Lambda can help simplify archiving data, reduce DynamoDB storage costs, and reduce code complexity. If the Lambda function returns an error, Lambda retries the batch until it processes successfully or the data expires. You can also configure Lambda to retry with a smaller batch, limit the number of retries, discard records after they become too old, and other options. Optionally, you can implement a DLQ mechanism, as described before. Finally, both OpenSearch Service and ElastiCache offer native backup mechanisms for recovery. For OpenSearch index reconstruction, you can either use these backups, perform a full scan of DynamoDB, or use DynamoDB to <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/S3DataExport.HowItWorks.html" rel="noopener" target="_blank">export to Amazon S3</a>. ElastiCache caches running Valkey, Redis OSS, or Serverless Memcached can back up their data by creating a snapshot, and you can use the backup to restore a cache or seed data to a new cache. The backup consists of the cache’s metadata, along with all of the data in the cache. All backups are written to Amazon S3. When it comes to rebuilding your DynamoDB-based architecture, the choice depends on factors like data volume, performance requirements, and recovery time objectives. For large DynamoDB tables, using DynamoDB export to S3 is generally more efficient than performing a full table scan. Note that a Lambda consumer for a DynamoDB stream doesn’t guarantee exactly once delivery and might lead to occasional duplicates. Make sure that your Lambda function code is <a href="https://docs.aws.amazon.com/wellarchitected/2025-02-25/framework/rel_prevent_interaction_failure_idempotent.html" rel="noopener noreferrer" target="_blank">idempotent</a> to prevent unexpected issues from arising because of duplicate processing.</li> 
 <li><strong>Amazon Kinesis Data Streams</strong> – To capture changes in DynamoDB, you can use <a href="https://aws.amazon.com/kinesis/data-streams/" rel="noopener noreferrer" target="_blank">Amazon Kinesis Data Streams</a>, which offers longer data retention (up to 365 days compared to 24 hours of DynamoDB Streams), support for up to 20 simultaneous consumers per shard with enhanced fan-out (compared to 2 with DynamoDB Streams), and the ability to combine DynamoDB changes with other data sources in the same stream for unified processing. However, there are important considerations regarding data consistency: unlike DynamoDB Streams, Kinesis Data Streams might occasionally produce duplicate records, requiring application-level deduplication logic. Additionally, while DynamoDB Streams guarantees in-order delivery of item-level modifications, with Kinesis Data Streams, applications need to implement ordering logic by comparing the <code>ApproximateCreationDateTime</code> timestamp attribute on stream records. For a detailed comparison of these change data capture options, see <a href="https://aws.amazon.com/blogs/database/choose-the-right-change-data-capture-strategy-for-your-amazon-dynamodb-applications/" rel="noopener noreferrer" target="_blank">Choose the right change data capture strategy for your Amazon DynamoDB applications</a>.</li> 
 <li><strong>Amazon Neptune</strong> – Neptune is <a href="https://docs.aws.amazon.com/neptune/latest/userguide/backup-restore-overview-fault-tolerance.html" rel="noopener noreferrer" target="_blank">fault tolerant</a> by design and is fed using Lambda functions, so the same error handling strategies can be implemented for Lambda in case of transient and persistent unavailability of the backend system. In terms of availability SLA, Neptune provides 99.9% SLA for <a href="https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-db-clusters.html" rel="noopener noreferrer" target="_blank">Multi-AZ DB instance, Multi-AZ DB cluster, and Multi-AZ graph</a>, whereas a single DB instance and Single-AZ graph provide 99.5% SLA. The Neptune <a href="https://docs.aws.amazon.com/neptune/latest/userguide/backup-restore.html#backup-restore-overview-backups" rel="noopener noreferrer" target="_blank">automated backups</a>, with a retention of up to 35 days, can be implemented to get PITR capability, to achieve an RPO of seconds or minutes and an RTO of hours. Optionally, you can use Neptune global databases to build a strong and reliable cross-Region disaster recovery solution, with lower RTO and less data loss (lower RPO) than traditional replication solutions.</li> 
 <li><strong>Amazon OpenSearch Service</strong> – OpenSearch Service comes with a 99.99% uptime SLA when <a href="https://aws.amazon.com/blogs/big-data/amazon-opensearch-service-under-the-hood-multi-az-with-standby/" rel="noopener noreferrer" target="_blank">Multi-AZ with standby</a> is implemented, and 99.9% when <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-multiaz.html#managedomains-za-no-standby" rel="noopener noreferrer" target="_blank">Multi-AZ is implemented without standby</a>. Although you can <a href="https://aws.amazon.com/blogs/big-data/unleash-the-power-of-snapshot-management-to-take-automated-snapshots-using-amazon-opensearch-service/" rel="noopener noreferrer" target="_blank">create snapshots of your OpenSearch indexes</a>, the best approach is implementing an <a href="https://aws.amazon.com/blogs/big-data/achieve-high-availability-in-amazon-opensearch-multi-az-with-standby-enabled-domains-a-deep-dive-into-failovers/" rel="noopener noreferrer" target="_blank">active-active architecture with failover</a>.</li> 
 <li><strong>Amazon ElastiCache for Valkey</strong> – <a href="https://aws.amazon.com/elasticache/sla/" rel="noopener noreferrer" target="_blank">ElastiCache for Valkey offers an availability SLA of 99.99%</a> when using a <a href="https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/AutoFailover.html" rel="noopener noreferrer" target="_blank">Multi-AZ configuration with automatic failover</a>. ElastiCache for Valkey also supports snapshot-based backups for data recovery. You can restore your data by creating a new ElastiCache for Valkey cache directly from a snapshot. When automatic backups are enabled, ElastiCache creates a backup of the cache on a daily basis, and maximum retention is 35 days. As previously described for OpenSearch Service, the recommended approach to maintain consistency is to rely on DynamoDB Streams and Lambda resiliency, including the DLQ mechanism, because the cache can be rebuilt from scratch.</li> 
</ul> 
<p>The core of the architecture that we consider in this use case is the product catalog implemented using Amazon DynamoDB, both the Enterprise Search system, and the Best Sellers features. The Best Sellers features are implemented respectively using Amazon OpenSearch Service and Amazon ElastiCache for Valkey, and are feed starting from the product catalog. This happens while the Social Media recommendations system is implemented using Amazon Neptune feed directly from the frontend interfaces through the <a href="https://aws.amazon.com/api-gateway/" rel="noopener" target="_blank">Amazon API Gateway</a> and AWS Lambda integration. One way to improve the architecture to make it more resilient is to introduce additional streaming/queueing layers like shown in the following diagram:</p> 
<p><img alt="" class="aligncenter wp-image-70095 size-full" height="1120" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/09/image-3-2.png" width="2142" /></p> 
<p>This way, you can store the incoming application frontend stream in Amazon Kinesis Data Streams, restore the product catalog and the social media recommendation engine with the latest available Point-in-time Restore, and reconcile by rebuilding only the last mile from the streaming mechanism, that acts effectively as a caching system.</p> 
<p>This architecture provides patterns for keeping data consistency across multiple purpose-built databases providing robust recovery mechanisms. The system can recover from failures, minimizing potential data loss in this mission-critical application.</p> 
<p>The approach is to rely on the specific service reliability and built-in features, such as automated backups. DLQs and streaming mechanisms can help make the entire architecture more robust and resilient, with more straightforward and more automated recovery procedures. In different scenarios, when you must maintain stronger data consistency among different backend systems, a <a href="https://en.wikipedia.org/wiki/Distributed_transaction" rel="noopener noreferrer" target="_blank">distributed transaction</a> system can be required, where applicable.</p> 
<p>Remember to regularly test and refine your recovery processes, simulating various failure scenarios to make sure that your system can maintain data consistency and recover seamlessly in real-world situations.</p> 
<h2>Conclusion</h2> 
<p>In this post, we discussed how aligning purpose-built databases require a thoughtful approach based on your specific application requirements. Whether you’re dealing with applications that have minimal consistency requirements or mission-critical systems demanding strict data alignment, AWS provides a comprehensive suite of tools and features to build resilient architectures.</p> 
<p>Our exploration reveals that understanding your consistency requirements is crucial before designing your recovery strategy. The foundation of data resilience can be built upon AWS built-in features such as DynamoDB Streams, PITR, and global tables, and implementing proper error handling mechanisms like DLQs helps maintain data consistency during failures. Remember that different scenarios might require different approaches, from straightforward independent recoveries to complex distributed transaction systems.As modern applications continue to evolve, the ability to effectively manage and recover multiple purpose-built databases will become increasingly important. The strategies and best practices outlined in this post provide a framework for addressing these challenges while making sure that your applications remain resilient and reliable.</p> 
<p>We’d love to hear what you think! If you have questions or suggestions, leave a comment.</p> 
<hr style="width: 120px;" /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="author name" class="size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/09/domenico.png" width="120" />
  </div> 
  <h3 class="lb-h4">Domenico di Salvia</h3> 
  <p><a href="https://www.linkedin.com/in/domenicodisalvia/" rel="noopener" target="_blank">Domenico</a> is a Senior Database Specialist Solutions Architect at AWS. In his role, Domenico works with customers in the EMEA region to provide guidance and technical assistance on database projects, helping them improve the value of their solutions when using or migrating to AWS, designing scalable, secure, performant, sustainable, cost-effective, and robust database architectures in the AWS Cloud.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/09/mttmss-200x300.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Marco Tamassia</h3> 
  <p><a href="https://www.linkedin.com/in/marcotamassia/" rel="noopener" target="_blank">Marco</a> is a Principal Technical Instructor based in Milan, Italy. He delivers a wide range of technical trainings to AWS customers across EMEA. Marco has a deep background as a database administrator for companies of all sizes, including AWS. This allows him to bring his database knowledge into the classroom, presenting real-world examples to his students.</p> 
 </div> 
</footer>
