---
title: "Implementing search on Amazon DynamoDB data using zero-ETL integration with Amazon OpenSearch service"
url: "https://aws.amazon.com/blogs/database/implementing-search-on-amazon-dynamodb-data-using-zero-etl-integration-with-amazon-opensearch-service/"
date: "Wed, 18 Feb 2026 20:19:55 +0000"
author: "Vamsi Krishna Ganti"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
<p>In this post, we show you&nbsp;how to implement search on <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> data using the <a href="https://aws.amazon.com/what-is/zero-etl/" rel="noopener noreferrer" target="_blank">zero-ETL</a> integration with <a href="https://aws.amazon.com/what-is/opensearch/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch </a>Service. You will learn how to add full-text search, fuzzy matching, and complex search queries to your application without building and maintaining data pipelines.</p> 
<p>Amazon DynamoDB is a great choice for applications demanding single-digit millisecond response times at scale and high-throughput operations. When combined with OpenSearch Service, applications gain powerful capabilities for complex search and analytics.Let’s explore a practical use case that demonstrates how this zero-ETL integration can enhance your application’s search capabilities.</p> 
<h2>Current state overview</h2> 
<p>AnyCompany, a fictional online marketplace specializing in eco-friendly and sustainable products, serves environmentally conscious consumers worldwide.&nbsp;Their application runs on an <a href="https://docs.aws.amazon.com/whitepapers/latest/optimizing-enterprise-economics-with-serverless/understanding-serverless-architectures.html" rel="noopener noreferrer" target="_blank">AWS serverless architecture</a> where <a href="https://docs.aws.amazon.com/lambda/latest/dg/welcome.html" rel="noopener noreferrer" target="_blank">AWS Lambda </a>functions handle e-commerce operations. The core transactional data such as product information and inventory details are stored in DynamoDB.</p> 
<p>The DynamoDB table layout for AnyCompany’s products includes the following attributes:</p> 
<table border="1px" cellpadding="10px" class="styled-table" width="100%"> 
 <tbody> 
  <tr> 
   <td><strong>Attribute</strong></td> 
   <td><strong>Example</strong></td> 
  </tr> 
  <tr> 
   <td>product_id (<strong>Partition Key</strong>)</td> 
   <td>Id_1</td> 
  </tr> 
  <tr> 
   <td>name</td> 
   <td>ECO T-Shirt</td> 
  </tr> 
  <tr> 
   <td>description</td> 
   <td>Organic CottonTee</td> 
  </tr> 
  <tr> 
   <td>price</td> 
   <td>100</td> 
  </tr> 
  <tr> 
   <td>rating</td> 
   <td>5</td> 
  </tr> 
  <tr> 
   <td>category</td> 
   <td>Activewear</td> 
  </tr> 
  <tr> 
   <td>brand</td> 
   <td>AnyCompany</td> 
  </tr> 
  <tr> 
   <td>tags</td> 
   <td>organic</td> 
  </tr> 
  <tr> 
   <td>specs</td> 
   <td>{“color”:”white”,”size”:”M”}</td> 
  </tr> 
  <tr> 
   <td>stock</td> 
   <td>48</td> 
  </tr> 
  <tr> 
   <td>created_at</td> 
   <td>2025-12-16T16:15:43.473864+0000</td> 
  </tr> 
  <tr> 
   <td>updated_at</td> 
   <td>2025-12-16T16:15:43.473864+0000</td> 
  </tr> 
 </tbody> 
</table> 
<p>For write operations, AnyCompany uses DynamoDB to maintain data consistency. When updating product information, the system uses the <code>product_id</code> as the partition key to perform product updates.&nbsp;AnyCompany’s product catalog expansion has led to more complex customer search patterns, requiring the system to handle variations, misspellings, semantic queries, and faceted browsing (allowing customers to filter and narrow down products using multiple attributes like price range, brand, ratings, and product categories).</p> 
<p>The following are search scenarios which are emerging as the company expands:</p> 
<ul> 
 <li>Multi-field search – Customers need to search across multiple fields such as product name, descriptions, categories and tags simultaneously.</li> 
 <li>Multi-range search – Customers want to filter products by multiple criteria, such as price range ($20-$50) and ratings (4-5 stars).</li> 
 <li>Discovery and Navigation –&nbsp;Customers want faceted browsing to progressively refine results (using filters like category, brands) and dynamic aggregations to understand their options (such as seeing product counts by category, brand, and rating distributions).</li> 
 <li>Fuzzy search and&nbsp;suggestions –&nbsp;Customers want their searches to work even when they make typos or spelling mistakes (for example, finding “reusable water bottle” when typing “reuseble botl”), while also receiving auto-complete suggestions as they type to help them find products more quickly.</li> 
 <li>Semantic search –&nbsp;Customers often use natural language descriptions rather than exact product names. For example, searching for “biodegradable kitchen storage containers” should return relevant eco-friendly food storage products.</li> 
</ul> 
<p>AWS offers <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/OpenSearchIngestionForDynamoDB.html" rel="noopener noreferrer" target="_blank">zero-ETL integration between DynamoDB and OpenSearch Service</a> to sync data between these two services. This integration enables you to complement your DynamoDB-based applications with search capabilities of OpenSearch Service to meet these search requirements, without building and maintaining custom data pipelines.</p> 
<p>In the following examples, we discuss how OpenSearch Service complements DynamoDB for these search scenarios at AnyCompany.</p> 
<h2>Multi-field search</h2> 
<p>Customer search patterns are inherently dynamic, with customers constantly seeking new ways to search products. At AnyCompany, customers want their search query to match across multiple product attributes simultaneously, looking for matches in product names, descriptions, categories, and tags to ensure they don’t miss relevant items.</p> 
<p>OpenSearch provides many features for customizing your search use cases and improving search relevance.&nbsp;For an exhaustive list, refer to <a href="https://opensearch.org/docs/latest/search-plugins/" rel="noopener noreferrer" target="_blank">Search features</a>.&nbsp;OpenSearch provides a search language called <a href="https://docs.opensearch.org/latest/query-dsl/" rel="noopener noreferrer" target="_blank"><em>query domain-specific language (DSL)</em></a> that you can use to search your data.</p> 
<p>The following is an example query that implements this functionality:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">GET products/_search
{
&nbsp;&nbsp;"query": {
&nbsp;&nbsp; &nbsp;"bool":{
&nbsp;&nbsp; &nbsp; &nbsp;"should": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp;]
&nbsp;&nbsp; &nbsp;}
&nbsp;&nbsp;},
&nbsp;&nbsp;"size":20
}</code></pre> 
</div> 
<h2>Multi range search</h2> 
<p>Building on the previous search functionality, AnyCompany’s customers often want to combine multiple filtering criteria, such as searching products within specific price ranges ($25–50) while considering specific categories and ratings.</p> 
<p>OpenSearch Service allows you to&nbsp;filter search results using different methods, each suited to specific scenarios. You can apply filters at the query level, using <code>Boolean</code> query clauses and <code>post_filter</code> and <code>aggregation</code> level filters.</p> 
<p>The following is an example query that implements this functionality:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">GET products/_search
{
&nbsp;&nbsp;"query": {
&nbsp;&nbsp; &nbsp;"bool":{
&nbsp;&nbsp; &nbsp; &nbsp;"should": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"multi_match": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"query": "Men's Wear",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"fields": ["name.ngram","description","category","tags"]
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp;],

&nbsp;&nbsp; &nbsp;}
&nbsp;&nbsp;},
&nbsp;&nbsp;"size":20
}</code></pre> 
</div> 
<h2>Discovery and navigation</h2> 
<p>At AnyCompany, customers want to discover products through an interactive interface rather than searching for specific items. For example, a customer browsing sustainable fashion might start with a search for “Men’s Wear” and want to understand the landscape of available products.&nbsp;They want to see available categories, brands offering sustainable options, common price points, and product ratings.&nbsp;The aggregated data&nbsp;typically displayed as filtering options in a sidebar, allowing customers to refine their search results intuitively.</p> 
<p><a href="https://docs.opensearch.org/latest/aggregations/" rel="noopener noreferrer" target="_blank">OpenSearch Aggregations</a> allow you to analyze data and extract statistics from it. You can apply metrics aggregation to perform simple calculations, bucket aggregation to&nbsp;categorize sets of documents as buckets, or pipeline aggregation to chain multiple aggregations together.</p> 
<p>The following is an example query that implements this functionality:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">GET products/_search
{
&nbsp;&nbsp;"query": {
&nbsp;&nbsp; &nbsp;"bool":{
&nbsp;&nbsp; &nbsp; &nbsp;"should": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"multi_match": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"query": "Men's Wear",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"fields": ["name.ngram","description","category","tags"]
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp;]
&nbsp;&nbsp; &nbsp;}
&nbsp;&nbsp;},
&nbsp;&nbsp;"aggs":&nbsp;{
&nbsp;&nbsp; &nbsp;"categories": {"terms":{"field":"category.keyword","size":10}},
&nbsp;&nbsp; &nbsp;"brands":{"terms":{"field":"brand.keyword","size":10}},
&nbsp;&nbsp; &nbsp;"colors":{"terms":{"field":"specs.color","size":10}},
&nbsp;&nbsp; &nbsp;"sizes":{"terms":{"field":"specs.size","size":10}},
&nbsp;&nbsp;"size":20
}</code></pre> 
</div> 
<h2>Fuzzy search and suggestions</h2> 
<p>At AnyCompany, customers frequently search for products with imperfect recall, often making typos while expecting real-time search suggestions that can predict their intent. For instance, typing “Omga” should tolerate typos and suggest relevant products like “OmegaWear” or “Omega Eco Essentials”. This requires a robust search system that handles spelling variations and typos while providing search suggestions to enhance the shopping experience.OpenSearch allows you to design autocomplete functionality that updates with each keystroke, provides relevant suggestions, and tolerates typos.</p> 
<p>The following is an example query that implements this functionality:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">GET products/_search
{
&nbsp;&nbsp;"suggest": {
&nbsp;&nbsp; &nbsp;"product_suggest": {
&nbsp;&nbsp; &nbsp; &nbsp;"prefix": "Om",
&nbsp;&nbsp; &nbsp; &nbsp;&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"size": 5
&nbsp;&nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;}
&nbsp;&nbsp;}
}</code></pre> 
</div> 
<h2>Semantic search</h2> 
<p>At AnyCompany, customers often express their product needs in conversational language, using natural phrases like “red beach hat for summer” instead of specific product names or categories. This intuitive way of searching requires a system that understands the semantic meaning behind customer queries, not just matching keywords.</p> 
<p>OpenSearch enables this natural search experience through <a href="https://docs.opensearch.org/latest/vector-search/" rel="noopener noreferrer" target="_blank">vector search</a>. The system converts both customer queries into vector embeddings – numerical representations that capture the semantic meaning of the text. This allows matching based on conceptual similarity rather than exact keyword matches.</p> 
<p>The following is an example query that implements this functionality:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">GET /products/_search
 {
&nbsp;&nbsp;&nbsp; "query": {
&nbsp;&nbsp;&nbsp; },
&nbsp;&nbsp;&nbsp; "aggs": {
&nbsp; &nbsp; &nbsp;&nbsp;&nbsp; "categories": { "terms": { "field": "category" }},
&nbsp; &nbsp; &nbsp;&nbsp;&nbsp; "brands": { "terms": { "field": "brand" }},
&nbsp; &nbsp; &nbsp;&nbsp;&nbsp; "avg_rating": { "avg": { "field": "rating" }}
&nbsp;&nbsp;&nbsp; },
&nbsp;&nbsp;&nbsp; "size": 20
}</code></pre> 
</div> 
<p>In the following sections, we show you how to implement a solution to support search functionality while maintaining your existing Amazon DynamoDB operations.</p> 
<h2>Solution overview</h2> 
<p>The following diagram illustrates the solution architecture.</p> 
<p><img alt="" class="alignnone wp-image-69090 size-full" height="1712" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/02/18/DBBLOG-4667-1v2.png" width="5884" /></p> 
<p>AnyCompany’s existing architecture follows a serverless pattern where <a href="https://aws.amazon.com/api-gateway/" rel="noopener noreferrer" target="_blank">Amazon API Gateway</a> routes customer requests to an AWS Lambda function. The function performs read and write operations on their DynamoDB table, which serves as the primary data store.</p> 
<p>To complement the existing architecture, AnyCompany implements the zero-ETL integration between DynamoDB and OpenSearch Service. The integration synchronizes data in two phases: <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Point-in-time-recovery.html" rel="noopener noreferrer" target="_blank">Point-in-Time Recovery (PITR)</a> exports existing DynamoDB data to Amazon S3 for the initial snapshot, while <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html" rel="noopener noreferrer" target="_blank">DynamoDB Streams</a> captures ongoing changes for near real-time updates to OpenSearch service.</p> 
<p>To enable <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/cfn-template.html" rel="noopener noreferrer" target="_blank">remote inference</a> needed for semantic search, you can connect OpenSearch to <a href="https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html" rel="noopener noreferrer" target="_blank">Amazon Bedrock’s Titan text embeddings</a> model using an <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ml-amazon-connector.html" rel="noopener noreferrer" target="_blank">OpenSearch ML Connector</a>.</p> 
<p>The OpenSearch Ingestion pipeline consumes these streams, transforms the data as needed, and synchronizes it to OpenSearch Service in near-real time. You specify an <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3) bucket as a dead-letter queue (DLQ) to capture any synchronization errors.</p> 
<p>To support search capabilities, the solution adds a new Lambda function that processes search requests using OpenSearch Service. The function uses the <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/configuration-samples.html" rel="noopener noreferrer" target="_blank">OpenSearch Service SDK</a> to create a client and execute search queries. This new function works alongside the existing write Lambda function, which continues to handle existing operations.</p> 
<p>This design allows AnyCompany to implement robust search features while preserving their e-commerce workload.</p> 
<h2>Implement a zero-ETL integration between Amazon DynamoDB and Amazon OpenSearch Service</h2> 
<p>In this section, we show how to setup a zero-ETL integration between DynamoDB and OpenSearch Service, including semantic search capabilities for enhanced search functionality.</p> 
<h2>Prerequisites</h2> 
<p>To set up this solution, you must have:</p> 
<ol> 
 <li>An <a href="https://signin.aws.amazon.com/signup?request_type=register" rel="noopener noreferrer" target="_blank">AWS account</a>.</li> 
 <li>Appropriate&nbsp;<a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity &amp; Access Management </a>(IAM)&nbsp;permissions to create and manage the following AWS resources: 
  <ol type="a"> 
   <li><a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithTables.Basics.html#WorkingWithTables.Basics.CreateTable" rel="noopener noreferrer" target="_blank">DynamoDB tables.</a></li> 
   <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/createupdatedomains.html" rel="noopener noreferrer" target="_blank">OpenSearch domains</a>.</li> 
   <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/creating-pipeline.html" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Ingestion (OSIS) pipelines. </a></li> 
   <li><a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html" rel="noopener noreferrer" target="_blank">CloudFormation templates</a>.</li> 
  </ol> </li> 
 <li><a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html" rel="noopener noreferrer" target="_blank">Amazon S3</a> General Purpose bucket for DynamoDB export and for DLQ event.</li> 
 <li>Amazon Bedrock with Amazon Titan Embeddings G1 enabled. For more information, see <a href="https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html" rel="noopener noreferrer" target="_blank">Access Amazon Bedrock foundation models</a>.</li> 
</ol> 
<h2>Implementation Steps</h2> 
<h3>Step 1: Create a DynamoDB table</h3> 
<p>For the solution walk through purposes, we create a new DynamoDB table named ‘Products’.&nbsp;If you’re implementing this solution&nbsp;with your existing DynamoDB table, you can skip this step.Run the following AWS CLI command to create the table:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb create-table \
--table-name Products \
--attribute-definitions \
AttributeName=product_id,AttributeType=S \
--key-schema \
AttributeName=product_id,KeyType=HASH \
--billing-mode PAY_PER_REQUEST \
--region us-east-1</code></pre> 
</div> 
<p>Wait until the table status changes from&nbsp;CREATING&nbsp;to&nbsp;ACTIVE. You can check the status with:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb describe-table \
--table-name Products \
--region us-east-1 \
--query 'Table.TableStatus'</code></pre> 
</div> 
<h3><strong>Step 2: Enable DynamoDB Streams and Point-in-Time Recovery (PITR)</strong></h3> 
<p>Run the following AWS CLI commands:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb update-table \
--table-name Products \
--stream-specification \
StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
--region us-east-1</code></pre> 
</div> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws dynamodb update-continuous-backups \
--table-name Products \
--point-in-time-recovery-specification \
PointInTimeRecoveryEnabled=true \
--region us-east-1</code></pre> 
</div> 
<h3>Step 3: Create Your OpenSearch Domain</h3> 
<p>Run the following AWS CLI command to create the domain:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws opensearch create-domain \
--domain-name demo \
--engine-version "OpenSearch_3.1" \
--cluster-config InstanceType=t3.small.search,InstanceCount=1 \
--ebs-options EBSEnabled=true,VolumeType=gp3,VolumeSize=10 \
--access-policies '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"*"},"Action":"es:*","Resource":"arn:aws:es:us-east-1:YOUR_ACCOUNT_ID&nbsp;:domain/demo/*"}]}' \
--advanced-security-options '{"Enabled":true,"InternalUserDatabaseEnabled":true,"MasterUserOptions":{"MasterUserName":"admin","MasterUserPassword": "YourStrongPassword123!"}}' \
--node-to-node-encryption-options Enabled=true \
--encryption-at-rest-options Enabled=true \
--domain-endpoint-options EnforceHTTPS=true \
--region us-east-1</code></pre> 
</div> 
<p><strong>Note:</strong></p> 
<ul> 
 <li>Replace&nbsp;<code>YOUR_ACCOUNT_ID</code>&nbsp;with your actual AWS account ID</li> 
 <li>Replace&nbsp;<code>YourStrongPassword123!</code>&nbsp;with a strong password (save this for logging into OpenSearch Dashboards)</li> 
</ul> 
<p><strong>Check the domain status:</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws opensearch describe-domain \
--domain-name demo \
--region us-east-1 \
--query 'DomainStatus.{Status:Processing,Endpoint:Endpoint }'</code></pre> 
</div> 
<p>Wait until the Processing field returns Status false, indicating the domain is active. Once active, note down the Endpoint (for API access) and navigate to the OpenSearch Dashboards URL (typically https://&lt;your-endpoint&gt;/_dashboards/) using the master username and password you created.</p> 
<h3>Step 4: Configure OpenSearch Security Settings</h3> 
<ul> 
 <li>Navigate to the OpenSearch Dashboards plugin for your OpenSearch Service domain. You can find the Dashboards endpoint on your domain dashboard on the <a href="https://us-east-1.console.aws.amazon.com/aos/home?region=us-east-1#opensearch" rel="noopener noreferrer" target="_blank">OpenSearch Service console</a>.</li> 
 <li>From the main menu choose&nbsp;<strong>Security</strong>, <strong>Roles</strong>, and select the <strong>ml_full_access</strong> role.</li> 
 <li>Choose <strong>Mapped users</strong>, <strong>Manage mapping</strong>.</li> 
 <li>Under <strong>Backend roles</strong>, add the ARN of the Lambda role that needs permission to call your domain. 
  <ul> 
   <li>arn:aws:iam::&lt;account-id&gt;:role/LambdaInvokeOpenSearchMLCommonsRole</li> 
  </ul> </li> 
 <li>Select <strong>Map</strong> and confirm the user or role shows up under <strong>Mapped users</strong>.</li> 
</ul> 
<p>This mapping enables the necessary permission for model integration. This role will be created in the next steps when you register the model id.</p> 
<div class="wp-caption alignnone" id="attachment_68406" style="width: 1930px;">
 <img alt="Figure 2: OpenSearch Security Settings" class="wp-image-68406 size-full" height="1080" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/26/RoleMapping-2.gif" width="1920" />
 <p class="wp-caption-text" id="caption-attachment-68406">Figure 2: OpenSearch Security Settings</p>
</div> 
<h3>Step 5: Set up the Bedrock&nbsp;Model Integration</h3> 
<p>With Remote inference, you can host your model inferences remotely on ML services, such as Amazon SageMaker AI and Amazon Bedrock, and connect them to Amazon OpenSearch Service with ML connectors.</p> 
<p>To ease the setup of remote inference, Amazon OpenSearch Service provides an <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a> template in the console.</p> 
<p>Open the <a href="https://console.aws.amazon.com/aos/home" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service console</a>.</p> 
<ul> 
 <li>In the left navigation pane, choose <strong>Integrations</strong>.</li> 
 <li>Choose&nbsp;<strong>Integrate with Amazon Titan Text Embeddings model through Amazon Bedrock</strong></li> 
 <li>Choose <strong>Configure domain</strong>, <strong>Configure public domain</strong>.</li> 
 <li>Provide the parameter values for “Amazon OpenSearch Endpoint” and “Model region”.</li> 
 <li>Check “<strong>I acknowledge that AWS CloudFormation might create IAM resources</strong>“</li> 
 <li>Choose&nbsp;<strong>Create stack</strong></li> 
 <li>After successful deployment of the stack, navigate to Outputs tab and note down the&nbsp;<strong>Model Id.</strong></li> 
</ul> 
<div class="wp-caption alignnone" id="attachment_68405" style="width: 1930px;">
 <img alt="Figure 3: Bedrock Model Integration" class="wp-image-68405 size-full" height="1080" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/26/ModelRegistration-2.gif" width="1920" />
 <p class="wp-caption-text" id="caption-attachment-68405">Figure 3: Bedrock Model Integration</p>
</div> 
<h3>Step 6: Set up the OpenSearch Ingest Pipeline</h3> 
<p>Create an <a href="https://docs.opensearch.org/latest/ingest-pipelines/" rel="noopener noreferrer" target="_blank">ingest pipeline</a> that uses the model in Amazon Bedrock to create embeddings from the input field <strong>description</strong> and map it to field named <strong>description_embedding</strong>.</p> 
<ul> 
 <li>Navigate to the OpenSearch Dashboards URL for your domain. You can find the URL on the domain’s dashboard in the <a href="https://us-east-1.console.aws.amazon.com/aos/home?region=us-east-1#opensearch/" rel="noopener noreferrer" target="_blank">OpenSearch Service console</a>.</li> 
 <li>Open the left navigation panel and choose&nbsp;<strong>DevTools</strong>.</li> 
 <li>Create your embedding pipeline: 
  <div class="hide-language"> 
   <pre><code class="lang-bash">PUT /_ingest/pipeline/bedrock-embedding-pipeline
{
  "description": "A text embedding pipeline",
  "processors": [
    {
      "text_embedding": {
        "model_id": "&lt;your_model_id&gt;", // Replace with the ModelId from previous step. 
        "field_map": {
          "description": "description_embedding"
        },
        "skip_existing": true
      }
    }
  ]
}</code></pre> 
  </div> </li> 
</ul> 
<div class="wp-caption alignnone" id="attachment_68408" style="width: 1930px;">
 <img alt="Figure 4: OpenSearch Ingest pipeline Creation" class="wp-image-68408 size-full" height="1080" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/26/IngestPipeline-2.gif" width="1920" />
 <p class="wp-caption-text" id="caption-attachment-68408">Figure 4: OpenSearch Ingest pipeline Creation</p>
</div> 
<h3>Step 7: Prepare your OpenSearch Index template</h3> 
<p>Before setting up the zero-ETL integration pipeline for your use case, analyze your search requirements and design appropriate index template for optimal performance.</p> 
<p><strong>Consider the following guidelines:</strong></p> 
<ul> 
 <li>Analyze access patterns and determine which fields need to be searchable and select appropriate field types. 
  <ul> 
   <li>Use keyword for exact matches like <code>product_id</code> and <code>category.</code></li> 
   <li>Use text with analyzers for full-text search capabilities.</li> 
   <li>Use date for timestamp fields.</li> 
  </ul> </li> 
 <li>To optimize your field selection, index only necessary fields to reduce processing overhead. 
  <ul> 
   <li>Implement custom analyzers for specialized search needs. For example, use n-gram analyzers for autocomplete functionality, which enables partial matching.</li> 
  </ul> </li> 
 <li>Determine your semantic search requirements. Embed those fields that are appropriate for your business use case.&nbsp;Configure vector field mappings with appropriate dimensions and algorithms based on your chosen embedding model and performance requirements.</li> 
</ul> 
<p>The following is an example of a mapping template:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">{
  "template": {
    "settings": {
      "index": {
        "number_of_shards": 1,
        "number_of_replicas": 1,
        "knn": true,
        "knn.algo_param.ef_search": 100,
        "default_pipeline": "bedrock-embedding-pipeline"
      },
      "analysis": {
        "analyzer": {
          "custom_analyzer": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": [
              "lowercase",
              "stop"
            ]
          },
          "ngram_analyzer": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": [
              "lowercase",
              "ngram_filter"
            ]
          }
        },
        "filter": {
          "ngram_filter": {
            "type": "ngram",
            "min_gram": 4,
            "max_gram": 5
          }
        }
      }
    },
    "mappings": {
      "properties": {
        "product_id": {
          "type": "keyword"
        },
        "name": {
          "type": "text",
          "analyzer": "custom_analyzer",
          "fields": {
            "keyword": {
              "type": "keyword"
            },
            "ngram": {
              "type": "text",
              "analyzer": "ngram_analyzer"
            },
            "suggest": {
              "type": "completion",
              "analyzer": "simple"
            }
          }
        },
        "description": {
          "type": "text",
          "analyzer": "custom_analyzer"
        },
        "price": {
          "type": "float"
        },
        "rating": {
          "type": "float"
        },
        "category": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword"
            }
          }
        },
        "brand": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword"
            }
          }
        },
        "tags": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword"
            }
          }
        },
        "specs": {
          "properties": {
            "color": {
              "type": "keyword"
            },
            "size": {
              "type": "keyword"
            },
            "weight": {
              "type": "float"
            },
            "dimensions": {
              "type": "keyword"
            }
          }
        },
        "stock": {
          "type": "integer"
        },
        "created_at": {
          "type": "date"
        },
        "updated_at": {
          "type": "date"
        },
        "description_embedding": {
          "type": "knn_vector",
          "dimension": 1536,
          "method": {
            "engine": "faiss",
            "space_type": "l2",
            "name": "hnsw",
            "parameters": {
              
            }
          }
        }
      }
    }
  }
} </code></pre> 
</div> 
<h3>Step 8: Setting up OpenSearch Ingestion Pipeline</h3> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/dynamodbv2/home" rel="noopener noreferrer" target="_blank">DynamoDB Service console</a>&nbsp;and&nbsp;Choose <strong>Integrations</strong>,&nbsp;create integration and choose Amazon OpenSearch.</li> 
 <li>Configure source (DynamoDB table) 
  <ul> 
   <li>Choose the DynamoDB table that will be the source of the integration from the dropdown.</li> 
   <li><strong>Enable Stream</strong></li> 
   <li><strong>Enable Export</strong> and configure S3 Bucket, S3 region</li> 
  </ul> </li> 
 <li><strong>Configure processor </strong>(Set up processors for data transformation)</li> 
 <li><strong>Configure sink</strong> (OpenSearch domain): 
  <ul> 
   <li><strong>Choose a domain</strong></li> 
   <li>Enter “products” as “Index Name”.</li> 
   <li>Choose Customized Mapping and use the template defined in&nbsp;Step 7.</li> 
   <li>Enable Dead-letter queues (DLQs) for offloading failed events and making them accessible for analysis. Provide the S3 bucket name and the region.</li> 
  </ul> </li> 
 <li><strong>Configure pipeline&nbsp;</strong> 
  <ul> 
   <li>Enter a unique pipeline name (across current AWS account in the current region).</li> 
   <li>Configure the pipeline capacity as per your requirement. A single Ingestion OpenSearch Compute Unit (OCU) represents billable compute and memory units. You are charged an hourly rate based on the number of OCUs used to run your data pipelines. As a baseline, one OCU can handle up to 2 MiB per second for typical workloads.&nbsp;Provide the minimum and maximum OCU capacity as per your workload.</li> 
   <li>OpenSearch Ingestion requires permissions to use other services on your behalf.&nbsp;Choose <strong>Create and use a new service role.&nbsp;</strong></li> 
   <li>Map the OSIS pipeline’s IAM role as a backend role in OpenSearch <strong>all_access</strong> role similar to <strong>Step 4. </strong>This enables secure, fine-grained access control for data ingestion.</li> 
  </ul> </li> 
 <li><strong>Review and create&nbsp;</strong></li> 
</ol> 
<ul> 
 <li>Review all the configurations.</li> 
 <li>Choose <strong>Preview pipeline </strong>and review the yaml file.</li> 
 <li>Choose <strong>Create Pipeline</strong> to create the pipeline.</li> 
</ul> 
<div class="wp-caption alignnone" id="attachment_68404" style="width: 1930px;">
 <img alt="Figure 5: OpenSearch Ingestion Pipeline Creation" class="wp-image-68404 size-full" height="1080" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/26/PipelineGif-2.gif" width="1920" />
 <p class="wp-caption-text" id="caption-attachment-68404">Figure 5: OpenSearch Ingestion Pipeline Creation</p>
</div> 
<h3>Validation</h3> 
<p>After setting up a zero-ETL integration, you can test and validate that the data is being replicated.</p> 
<ol> 
 <li>Create sample items in DynamoDB. The following is a sample shell script to create items: 
  <div class="hide-language"> 
   <pre><code class="lang-bash">#!/bin/bash

for i in {1..10}; do
  colors=("red" "white" "green" "blue" "black")
  sizes=("XS" "S" "M" "L" "XL")
  products=("Hoodie" "Jacket" "Pants" "Shorts" "Sweater" "Joggers" "Tank Top" "Polo Shirt" "Vest" "Leggings")
  categories=("Activewear" "Men's Wear" "Women's Wear" "Loungewear" "Outerwear")
  brands=("AnyCompany1" " AnyCompany2" " AnyCompany3" " AnyCompany4" " AnyCompany5" " AnyCompany6")
  features=("Sustainably crafted" "Eco-friendly materials" "Ethically produced" "Premium quality" "Organic materials")

  color=${colors[$((RANDOM % 5))]}
  features=${features[$((RANDOM % 5))]}
  size=${sizes[$((RANDOM % 5))]}
  category=${categories[$((RANDOM % 5))]}
  brand=${brands[$((RANDOM % 6))]}
  product_name=${products[$((i - 1))]}
  timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  aws dynamodb put-item \
    --table-name Products \
    --region us-east-1 \
    --item "{
      \"product_id\": {\"S\": \"product_$i\"},
      \"name\": {\"S\": \"$brand $product_name\"},
      \"description\": {\"S\": \"$features $product_name - $category\"},
      \"price\": {\"N\": \"$((100 + RANDOM % 400)).99\"},
      \"rating\": {\"N\": \"4.$((RANDOM % 10))\"},
      \"category\": {\"S\": \"$category\"},
      \"brand\": {\"S\": \"$brand\"},
      \"tags\": {\"L\": [{\"S\": \"organic\"}, {\"S\": \"sustainable\"}, {\"S\": \"eco-friendly\"}]},
      \"specs\": {\"M\": {
        \"color\": {\"S\": \"$color\"},
        \"size\": {\"S\": \"$size\"},
        \"weight\": {\"N\": \"0.$((RANDOM % 9 + 1))\"},
        \"dimensions\": {\"S\": \"$((RANDOM % 30 + 10))x$((RANDOM % 30 + 10))x$((RANDOM % 10 + 1))\"}
      }},
      \"stock\": {\"N\": \"$((RANDOM % 200))\"},
      \"created_at\": {\"S\": \"$timestamp\"},
      \"updated_at\": {\"S\": \"$timestamp\"}
    }" &amp;&amp; echo "✓ Created: product_id=$i"
done

echo "Done!"
</code></pre> 
  </div> </li> 
 <li>Navigate to the OpenSearch Dashboards URL for your domain. You can find the URL on the domain’s dashboard in the OpenSearch Service console.</li> 
 <li>Open the left navigation panel and choose <strong>Dev Tools</strong>.</li> 
 <li>Verify data flow to OpenSearch. 
  <div class="hide-language"> 
   <pre><code class="lang-bash">//search all
GET products/_search
{
  "query": {
    "match_all": {}
  }
}
</code></pre> 
  </div> </li> 
 <li>Run the semantic use case query to validate your semantic requirements. 
  <div class="hide-language"> 
   <pre><code class="lang-bash">GET products/_search
{
  "query": {
    "neural": {
      "description_embedding": {
        "query_text": "sports",
        "model_id": "&lt;update the model_id obtained from Step 5&gt;"
      }
    }
  }
}
</code></pre> 
  </div> </li> 
</ol> 
<p>You can test all other use cases using the queries provided earlier in the post.</p> 
<div class="wp-caption alignnone" id="attachment_68407" style="width: 1930px;">
 <img alt="Figure 6: Data Validation" class="wp-image-68407 size-full" height="1080" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/26/DataValidation-2.gif" width="1920" />
 <p class="wp-caption-text" id="caption-attachment-68407">Figure 6: Data Validation</p>
</div> 
<p>After you complete the pipeline setup and validate the data synchronization, refactor your application code to route search queries to OpenSearch Service while keeping transactional operations in DynamoDB.</p> 
<p>The specific implementation depends on your application architecture and programming language. For instructions on querying OpenSearch Service using the AWS SDKs, refer to <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/configuration-samples.html" rel="noopener noreferrer" target="_blank">Using the AWS SDKs to interact with Amazon OpenSearch Service.</a></p> 
<h2>Best practices</h2> 
<p>When implementing zero-ETL integration between DynamoDB and OpenSearch Service, consider the following <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-integration-opensearch.html" rel="noopener noreferrer" target="_blank">best practices</a>:</p> 
<ul> 
 <li>Use DynamoDB streams as the OSIS Pipeline source option, which captures only new or updated records without requiring a full initial snapshot. Once the pipeline flow is tested with streaming data, use export option to migrate the complete data.</li> 
 <li>Use an Amazon VPC to enforce data flow through your OpenSearch Ingestion pipelines within your network boundaries, rather than over the public internet.</li> 
 <li>Optimize your embedding generation ingest pipeline by enabling&nbsp;<a href="https://docs.opensearch.org/latest/ingest-pipelines/processors/text-embedding/" rel="noopener noreferrer" target="_blank">skip_existing</a> flag to prevent redundant API calls&nbsp;for fields that already contain embeddings.</li> 
 <li>Enable <a href="http://aws.amazon.com/cloudwatch" rel="noopener noreferrer" target="_blank">Amazon CloudWatch logs</a> for your OpenSearch Service domain and OpenSearch Ingestion pipeline. This provides visibility into the entire data flow and helps identify issues quickly. Configure logging at the domain level to capture search queries, indexing operations, and cluster health metrics. For OpenSearch Ingestion pipelines, enable detailed logging to monitor data transformation and ingestion processes.</li> 
 <li>Configure a DLQ for your ingestion pipeline. When documents fail to process due to mapping errors, transformation issues, or temporary service problems, the DLQ captures these failed records for later analysis and reprocessing. This prevents data loss and provides a recovery mechanism.</li> 
 <li>Enable encryption at rest&nbsp;using <a href="https://docs.aws.amazon.com/kms/latest/developerguide/overview.html" rel="noopener noreferrer" target="_blank">AWS Key Management Service (AWS KMS)</a> for both DynamoDB tables and OpenSearch Service domains to maintain full control over your encryption keys and meet compliance requirements.</li> 
 <li>Enable DynamoDB deletion protection&nbsp;on your tables to prevent accidental deletion of critical data, providing an additional safety layer for your production workloads.</li> 
 <li>Implement least privilege access&nbsp;by creating specific IAM roles and policies for each component of the integration pipeline, ensuring that services have only the permissions they need to function.</li> 
 <li>For OpenSearch Service domain sizing recommendations including shard sizing, instance types, and deployment configurations, refer to the <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/sizing-domains.html" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service domain sizing guide</a>.</li> 
</ul> 
<h2>Monitoring and alerting</h2> 
<p>OpenSearch Ingestion publishes metrics that are useful to monitor your pipeline performance. When monitoring your pipeline health, focus on the following key areas:</p> 
<ul> 
 <li>Authentication issues that might block data ingestion.</li> 
 <li>Document processing failures that could lead to data loss.</li> 
 <li>Capacity bottlenecks that can slow down data processing.</li> 
 <li>Performance degradation that affects your application’s response times.</li> 
 <li>For Bedrock-specific monitoring, pay special attention to model invocations latency, throttling and token usage metrics.</li> 
</ul> 
<p>By actively monitoring these areas, you can maintain a reliable data flow from DynamoDB to OpenSearch Service and ensure your search functionality remains responsive. Set up CloudWatch alerts for early detection of these issues, so you can take proactive action before they impact your users.</p> 
<p>For more information and best practices for <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/monitoring-pipelines.html" rel="noopener noreferrer" target="_blank">logging and monitoring</a>, refer to <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/osis-best-practices.html" rel="noopener noreferrer" target="_blank">Best practices for Amazon OpenSearch Ingestion</a>&nbsp;and&nbsp;<a href="https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring.html" rel="noopener noreferrer" target="_blank">Amazon Bedrock monitoring </a>documentation.</p> 
<h2>Cost Considerations</h2> 
<p>When planning your DynamoDB to OpenSearch Service integration using OpenSearch Ingestion, analyze your DynamoDB write throughput through Amazon CloudWatch metrics. Your DynamoDB table’s write activity generates records in DynamoDB Streams, which the OpenSearch Ingestion pipeline consumes. Understanding your minimum, maximum, and average write request units (WRUs) helps you right-size your OpenSearch Ingestion pipeline. As of this writing, an OpenSearch Ingestion pipeline can scale from 1 to 96 OpenSearch Compute Units (OCUs). As a baseline, one OCU can handle up to 2 MiB per second for typical workloads which may vary depending on your data size, pipeline transformations, and OpenSearch Service domain size and index/shard strategy.</p> 
<p>For example, if your table processes 20,000 WRUs per second, you need a minimum of 10 OCUs to handle this throughput efficiently. Your total costs include DynamoDB Streams charges ($0.02 per 100,000 read requests), OpenSearch Service cluster costs (based on instance types and storage), and OpenSearch Ingestion pipeline costs (priced per OCU-hour) and the regular DynamoDB costs. When incorporating Amazon Bedrock for remote inference in your DynamoDB to OpenSearch pipeline, evaluate which fields require embeddings and estimate the volume of text data you’ll process. Calculate your expected token usage based on the number of records, the size of text fields, and your update frequency. The cost structure follows a pay-per-token model, which varies by the foundation model you select.</p> 
<h2>Clean up</h2> 
<p>To avoid incurring ongoing charges, delete or disable the resources you created in this solution. Follow these steps in order to clean up your AWS environment:</p> 
<ul> 
 <li>Delete the CloudFormation <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html" rel="noopener noreferrer" target="_blank">stack</a>: 
  <ul> 
   <li>Open the CloudFormation console.</li> 
   <li>Select the stack you created for model access.</li> 
   <li>Choose <strong>Delete</strong> and confirm the deletion.</li> 
  </ul> </li> 
 <li>Remove OpenSearch resources: 
  <ul> 
   <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/gsgdeleting.html" rel="noopener noreferrer" target="_blank">Delete the OSIS pipeline</a> by opening the Amazon OpenSearch Ingestion console and removing the pipeline.</li> 
   <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/gsgdeleting.html" rel="noopener noreferrer" target="_blank">Delete the OpenSearch domain</a> through the Amazon OpenSearch Service console.</li> 
  </ul> </li> 
 <li><a href="https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_DeleteTable.html" rel="noopener noreferrer" target="_blank">Delete the DynamoDB table (Products).</a></li> 
 <li>Empty the S3 bucket used for the dead letter queue (DLQ) and DynamoDB export.</li> 
</ul> 
<p>After completing these steps, verify in each service console that all resources have been properly deleted or disabled.</p> 
<h2>Conclusion</h2> 
<p>In this post, we showed you&nbsp;how to implement search on <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> data using the <a href="https://aws.amazon.com/what-is/zero-etl/" rel="noopener noreferrer" target="_blank">zero-ETL</a> integration with <a href="https://aws.amazon.com/what-is/opensearch/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch </a>Service.</p> 
<p>Through the AnyCompany example, we demonstrated how this integration addresses real-world search challenges—from handling customer typos and semantic queries to enabling faceted browsing and multi-field searches.</p> 
<p>Get started by exploring the&nbsp;<a href="https://docs.aws.amazon.com/opensearch-service/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service documentation</a>&nbsp;and <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/OpenSearchIngestionForDynamoDB.html" rel="noopener noreferrer" target="_blank">the&nbsp;DynamoDB zero-ETL integration guide</a>&nbsp;to begin implementing these features in your own environment.</p> 
<hr /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <h3 class="lb-h4">Varun Sharma</h3> 
  <p><a href="https://www.linkedin.com/in/varun2235/" rel="noopener" target="_blank">Varun</a> is a Senior Delivery Consultant working with the AWS Professional Services team.</p> 
 </div> 
 <div class="blog-author-box"> 
  <h3 class="lb-h4">Vamsi Krishna Ganti</h3> 
  <p><a href="https://www.linkedin.com/in/vamsi-krishna-ganti-34588030/" rel="noopener" target="_blank">Vamsi</a> is a Senior Delivery Consultant working with the AWS Professional Services team.</p> 
 </div> 
 <div class="blog-author-box"> 
  <h3 class="lb-h4">Sudhansu Mohapatra</h3> 
  <p><a href="https://www.linkedin.com/in/sudhansu-mohapatra-83a51811a/" rel="noopener" target="_blank">Sudhansu</a> is a Delivery Consultant working with the AWS Professional Services team.</p> 
 </div> 
</footer>
