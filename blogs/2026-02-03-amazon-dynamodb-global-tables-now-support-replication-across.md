---
title: "Amazon DynamoDB global tables now support replication across AWS accounts"
url: "https://aws.amazon.com/blogs/database/amazon-dynamodb-global-tables-now-support-replication-across-aws-accounts/"
date: "Tue, 03 Feb 2026 20:47:35 +0000"
author: "Robert McCauley"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/02/03/DBBLOG-5196-1.png" /></p> 
<p><a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> global tables is a fully managed, multi-Region, and multi-active database feature that provides seamless data replication and fast local read and write performance for globally scaled applications.</p> 
<p>Today, we’re announcing <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables-MultiAccount.html" rel="noopener noreferrer" target="_blank">multi-account global tables for Amazon DynamoDB</a>, which let you replicate DynamoDB table data across multiple AWS accounts and AWS Regions. This feature adds account-level isolation to global tables, so you can replicate DynamoDB table data across multiple AWS accounts and Regions for stronger isolation and resiliency.</p> 
<p>With this feature, DynamoDB now supports two global tables models, each designed for different architectural patterns:</p> 
<ul> 
 <li><strong>Same-account global tables</strong> – Replicas are created and managed within a single AWS account</li> 
 <li><strong>Multi-account global tables</strong> – Replicas are deployed across multiple AWS accounts while participating in a shared replication group</li> 
</ul> 
<p>Both models support fast local writes, asynchronous replication, and last-writer-wins conflict resolution. However, they differ in how accounts, permissions, encryption, and table governance are managed. Multi-account global tables currently support <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html#V2globaltables_HowItWorks.consistency-modes" rel="noopener noreferrer" target="_blank">multi-Region eventual consistency (MREC)</a> only.</p> 
<p>In this post, we show you how to create and configure a multi-account global table, and introduce use cases highlighting the value of using this feature.</p> 
<h2>Enhanced disaster recovery architecture</h2> 
<p>Multi-account global tables transform how you can architect disaster recovery solutions. By distributing your data across multiple AWS accounts, you can have additional layers of isolation to limit the impact of misconfigurations, security incidents, or account-level issues.Consider a scenario where your primary application runs in Account1 (us-east-1) and your disaster recovery environment operates in Account2 (us-west-2). With multi-account global tables, both accounts maintain synchronized copies of your critical data, enabling rapid failover without complex data migration procedures.</p> 
<h2>Organizational compliance and cost attribution</h2> 
<p>Many enterprises operate with multiple AWS accounts for organizational, security, or compliance reasons. Multi-account global tables help these organizations maintain data consistency across their distributed infrastructure while respecting existing compliance boundaries, guardrails, and governance models.For example, a financial services company might maintain separate accounts for different business units or regulatory environments. Multi-account global tables allow these units to share critical reference data while maintaining the isolation required by their compliance frameworks. In addition, the costs for each Regional replica are aligned to AWS accounts that might be managed by separate business units.</p> 
<p>For more information on multi-account strategies, refer to <a href="https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/aws-account-management-and-separation.html" rel="noopener noreferrer" target="_blank">AWS account management and separation</a> and <a href="https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/benefits-of-using-multiple-aws-accounts.html" rel="noopener noreferrer" target="_blank">Benefits of using multiple AWS accounts</a>.</p> 
<h2>How DynamoDB multi-account global tables work</h2> 
<p>Multi-account global tables use permissions defined in <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/access-control-resource-based.html" rel="noopener noreferrer" target="_blank">resource-based policies</a> to indicate which other accounts can join the replication group, and to allow data to be replicated.</p> 
<p>Each replica must reside in a separate AWS account and a separate Region. For a multi-account global table with <em>N</em> replicas, you must have <em>N</em> accounts in <em>N</em> separate Regions.</p> 
<p>You can begin with an existing, non-empty single Region table, and then add a replica table in another Region and account. The system will copy existing items into the new table. When both tables are synchronized, you will see each table’s status as ACTIVE.</p> 
<p>Multi-account global tables publish the <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/metrics-dimensions.html#ReplicationLatency" rel="noopener noreferrer" target="_blank">ReplicationLatency</a> metric to <a href="http://aws.amazon.com/cloudwatch" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a>. This metric tracks the elapsed time between when an item is written to a replica table and when that item appears in another replica in the global table. You can monitor this metric to understand how quickly items are replicated to remote Regions.</p> 
<h2>Multi-account global tables: Settings replication behavior</h2> 
<p>When creating a multi-account global table, you must set <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html#V2globaltables_HowItWorks.setting-synchronization" rel="noopener noreferrer" target="_blank">GlobalTableSettingsReplication</a> to ENABLED for each Regional replica. This means configuration changes made in one Region will propagate automatically to other Regions that participate in the global table.</p> 
<p>For the source table, you can enable settings replication after table creation. This supports the scenario where a table is originally created as a Regional table and later upgraded to a multi-account global table.</p> 
<p>Refer to <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html#V2globaltables_HowItWorks.setting-synchronization" rel="noopener noreferrer" target="_blank">Settings synchronization</a> for a list of synchronized and non-synchronized replica settings.</p> 
<h2>Solution overview</h2> 
<p>In this post, we provide a high-level summary of steps required to use multi-account global tables. For a detailed tutorial, refer to <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_MA.tutorial.html" rel="noopener noreferrer" target="_blank">Tutorials: Creating global tables</a>.</p> 
<p>For our example, we use two accounts: ACCOUNT1 in REGION1 and ACCOUNT2 in REGION2.</p> 
<p>We can create the Amazon Resource Names (ARNs) for each table replica in advance as follows, assuming the new table is called myTable:</p> 
<ul> 
 <li><strong>ACCOUNT1_TABLE_ARN</strong>: “arn:aws:dynamodb:REGION1:ACCOUNT1:table/myTable”</li> 
 <li><strong>ACCOUNT2_TABLE_ARN</strong>: “arn:aws:dynamodb:REGION2:ACCOUNT2:table/myTable”</li> 
</ul> 
<ol> 
 <li>Create a DynamoDB table in REGION1. You can add items to the table or use an existing single-Region table that has items. For this post, we name the table myTable.</li> 
 <li>Set the table’s GlobalTableSettingsReplication: ENABLED.</li> 
</ol> 
<p>The following screenshot shows this setting on the DynamoDB console.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/02/03/DBBLOG-5196-2.png" /></p> 
<p>If you are using the <a href="http://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface</a> (AWS CLI), you can also indicate this within the create-table command by adding –global-table-settings-replication ENABLED.</p> 
<ol> 
 <li>Add a resource policy to the table, with the following two statements:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
    "Version": "2012-10-17",
    "Statement": [
       {
         "Sid": " AllowTrustedAccountsToJoinThisGlobalTable",
         "Effect": "Allow",
         "Principal": {
             "AWS": [&lt;ACCOUNT2&gt;]
         },
         "Action": "dynamodb:AssociateTableReplica",
         "Resource": &lt;ACCOUNT1_TABLE_ARN&gt;
       },
       {
         "Sid": "AllowReplication",
         "Effect": "Allow",
         "Principal": {
             "Service": "replication.dynamodb.amazonaws.com"
         },
         "Action": [
                      "dynamodb:ReadDataForReplication",
                      "dynamodb:WriteDataForReplication",
                      "dynamodb:ReplicateSettings"
                    ],                
         "Resource": &lt;ACCOUNT1_TABLE_ARN&gt;,
         "Condition": {
            "StringEquals": {
               "aws:SourceAccount": [&lt;ACCOUNT1&gt;, &lt;ACCOUNT2&gt;],
               "aws:SourceArn": [&lt;ACCOUNT1_TABLE_ARN&gt;, 
                                 &lt;ACCOUNT2_TABLE_ARN&gt;]
            }          
         }
       }
    ]
}</code></pre> 
</div> 
<p>The Condition section of these policies is required so the DynamoDB <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables_MA_security.html#globaltables_MA_service_principal" rel="noopener noreferrer" target="_blank">service principal</a>&nbsp;will have permissions to replicate data amongst the tables you specify. You can add additional accounts and ARNs to the resource-based policy if you need to expand your global table to additional accounts and Regions.</p> 
<ol start="2"> 
 <li>Create a DynamoDB table in ACCOUNT2 and REGION2 with the following settings: 
  <ul> 
   <li>GlobalTableSettingsReplication: ENABLED</li> 
   <li>Include a resource policy with the following format:</li> 
  </ul> </li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
    "Version": "2012-10-17",
    "Statement": [
       {
         "Sid": "AllowReplication",
         "Effect": "Allow",
         "Principal": {
             "Service": "replication.dynamodb.amazonaws.com"
         },
         "Action": [
                      "dynamodb:ReadDataForReplication",
                      "dynamodb:WriteDataForReplication",
                      "dynamodb:ReplicateSettings"
         ],
         "Resource": &lt;ACCOUNT2_TABLE_ARN&gt;,
         "Condition": {
            "StringEquals": {
               "aws:SourceAccount": [&lt;ACCOUNT1&gt;, &lt;ACCOUNT2&gt;],
               "aws:SourceArn": [&lt;ACCOUNT1_TABLE_ARN&gt;, &lt;ACCOUNT2_TABLE_ARN&gt;]
            }
         }
       }
    ]
}</code></pre> 
</div> 
<p>You can also accomplish this step on the DynamoDB console. Choose the <strong>Create table</strong> dropdown menu and choose <strong>Create cross-account global table replica</strong>.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/02/03/DBBLOG-5196-3.png" /></p> 
<p>The following screenshot shows the configuration details required.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/02/03/DBBLOG-5196-4.png" /></p> 
<h2>Use cases</h2> 
<p>One type of disaster planning is the scenario of a malicious actor gaining full control of Account1. Should this happen, the owner of Account2 can halt replication by updating their table’s resource policy to deny replication actions. If the table has <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/PointInTimeRecovery_Howitworks.html" rel="noopener noreferrer" target="_blank">point-in-time recovery</a> enabled, you can perform an <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/S3DataExport.HowItWorks.html" rel="noopener noreferrer" target="_blank">incremental export</a> to <a href="http://aws.amazon.com/s3" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3) to get a snapshot of all writes from the last 24 hours in JSON format. Then, you can review the <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/S3DataExport.Output.html" rel="noopener noreferrer" target="_blank">new and old</a> images of any items changed, to see the original state of any items that could have been maliciously altered. This will be flagged as an abnormal state for a global table, so <a href="https://aws.amazon.com/contact-us/" rel="noopener noreferrer" target="_blank">AWS Support</a> might reach out to you to verify why replication has stopped.</p> 
<p>Another use case is when you want to move a table between AWS accounts. At the time of writing, multi-account global tables don’t support same-Region replication, so a series of steps must be performed, temporarily involving another Region. The high-level steps are as follows:</p> 
<ol> 
 <li>Configure your application to be able to switch the AWS account and Region used for authentication to DynamoDB.</li> 
 <li>Use the steps covered in this post to: 
  <ol type="a"> 
   <li>Add a resource policy to the table in Account1, Region1.</li> 
   <li>Create a linked replica table in Account2, Region2.</li> 
   <li>Wait 24 hours. You cannot delete the original table replica until 24 hours have passed.</li> 
  </ol> </li> 
 <li>Adjust your application to use the DynamoDB table in Account2, Region2.</li> 
 <li>Delete the table replica in Account1, Region1. (The system will automatically copy over all pending writes before removing the replica.)</li> 
 <li>Using Account2, call <a href="https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_UpdateTable.html" rel="noopener noreferrer" target="_blank">update-table</a> to request a new same-account replica be added in Region1.</li> 
 <li>Check the table status. When it returns to ACTIVE, your table replica in Account2, Region1 is ready.</li> 
 <li>Switch the application to use Account2, Region1.</li> 
 <li>(Optional) Delete the table replica in Account2, Region2.</li> 
</ol> 
<h2>Summary</h2> 
<p>DynamoDB global tables now support replication across multiple AWS accounts. This enhances resiliency though account-level isolation, supports tailored security and data-perimeter controls, enables alignment of workloads by business unit or environment, and simplifies governance requirements. To learn more, refer to <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html" rel="noopener noreferrer" target="_blank">Global tables – multi-active, multi-Region replication</a> and <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/disaster-recovery-resiliency.html" rel="noopener noreferrer" target="_blank">Resilience and disaster recovery in Amazon DynamoDB</a>. Please let us know your feedback in the comments section.</p> 
<hr /> 
<h2>About the author</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Robert McCauley" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/02/03/DBBLOG-5196-5.png" width="120" />
  </div> 
  <h3 class="lb-h4">Robert McCauley</h3> 
  <p><a href="https://www.linkedin.com/in/robm26/" rel="noopener" target="_blank">Robert</a> is an Amazon DynamoDB Specialist Solutions Architect based out of Boston. He began his Amazon career in 2012 as a SQL developer at Amazon Robotics, followed by a stint as an Alexa Skills solutions architect, before joining AWS.</p> 
 </div> 
</footer>
