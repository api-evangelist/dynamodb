---
title: "Simplify cross-account stream processing with AWS Lambda and Amazon DynamoDB"
url: "https://aws.amazon.com/blogs/database/simplify-cross-account-stream-processing-with-aws-lambda-and-amazon-dynamodb/"
date: "Mon, 09 Feb 2026 19:40:01 +0000"
author: "Lee Hannigan"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
<p>Organizations often use a multi-account architecture for security and isolation. However, with your <a href="https://aws.amazon.com/dynamodb" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> tables now in one account, you might need to process their stream events in another. Until recently, this meant routing through <a href="https://aws.amazon.com/kinesis" rel="noopener noreferrer" target="_blank">Amazon Kinesis Data Streams</a> or building custom relay infrastructure with cross-account <a href="https://aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management</a> (IAM) roles, adding unwanted complexity. <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/access-control-resource-based.html" rel="noopener noreferrer" target="_blank">Resource-based policies</a> for <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html" rel="noopener noreferrer" target="_blank">Amazon DynamoDB Streams</a> now helps you avoid these workarounds. Your <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda </a>functions can directly consume streams across accounts with no custom infrastructure required.</p> 
<p>DynamoDB is a serverless, fully managed, distributed NoSQL database with single-digit millisecond performance at scale. You can build modern, high-performance applications without managing infrastructure. One of its key features is DynamoDB Streams, which captures data changes in near real time. This capability supports use cases such as audit logging, search indexing, cross-Region replication, anomaly detection, and real-time analytics.</p> 
<p>Lambda is a serverless compute service you can use to run code without provisioning or managing servers. Lambda integrates with DynamoDB Streams, so you can automatically trigger functions in response to table updates. This integration is helpful for use cases like data replication, materialized views, analytics pipelines, and event-driven architectures.</p> 
<p>In this post, we explore how to use resource-based policies with DynamoDB Streams to enable cross-account Lambda consumption. We focus on a common pattern where application workloads live in isolated accounts, and stream processing happens in a centralized or analytics account.</p> 
<h2>Benefits of cross-account Lambda with DynamoDB streams</h2> 
<p>With resource-based policies for DynamoDB streams, you can grant a Lambda function in one AWS account direct access to read from a DynamoDB stream in another account. No custom relay infrastructure is required.</p> 
<p>With this feature, you can now simplify multi-account event-driven architectures, improve security, and reduce operational burden. Lambda manages the ingestion, filtering, delivery, retries, and error handling just as it does with same account <a href="https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html" rel="noopener noreferrer" target="_blank">event source mappings</a>.</p> 
<p>There are many reasons you might want to process DynamoDB stream events in a different AWS account, such as if you’re operating a SaaS offering, running a centralized analytics pipeline, or managing isolated environments for development, staging, and production. Resource-based policy support for DynamoDB streams makes it straightforward to implement these patterns while maintaining strong access control.</p> 
<p>This capability enables several architectural patterns:</p> 
<ul> 
 <li><strong>Centralized data processing</strong> – Route streams from multiple application accounts to a central analytics or data lake account where Lambda functions perform aggregation, transformation, and loading into your data warehouse.</li> 
 <li><strong>Shared services</strong> – Build reusable audit logging, compliance monitoring, or notification services in a dedicated account that consume streams from tables across your organization.</li> 
 <li><strong>Multi-tenant architectures</strong> – Allow tenant-specific Lambda functions in isolated accounts to process data from a centralized DynamoDB table, maintaining security boundaries while supporting event-driven workflows.</li> 
</ul> 
<h2>Solution overview</h2> 
<p>The cross-account access model uses two components:</p> 
<ul> 
 <li><strong>Resource-based policy on the DynamoDB stream (in the source account)</strong> – Grants permission to the Lambda execution role in the consuming account</li> 
 <li><strong>IAM execution role (in the consuming account)</strong> – Allows the Lambda function to read from the stream</li> 
</ul> 
<p>When you configure an event source mapping between a Lambda function and a cross-account DynamoDB stream, access is granted using the combined permissions of the stream’s resource policy and the Lambda function’s execution role. The stream’s resource-based policy authorizes the Lambda execution role, and the execution role provides the specific permissions needed to read and process stream records.</p> 
<p>To illustrate how this works, consider a SaaS provider that offers document processing services to enterprise customers. Each customer operates in a dedicated AWS account for improved isolation and security. When a document is processed, the tenant writes a record to a DynamoDB table with DynamoDB streams enabled. The SaaS offering team wants to aggregate these records in a central AWS account for billing and analytics. With cross-account Lambda and DynamoDB streams, this integration becomes straightforward and fully managed.</p> 
<p>The following diagram illustrates the architecture.</p> 
<p><img alt="Architecture diagram" class="aligncenter size-full wp-image-68614" height="886" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/29/5323-image-1.png" width="1377" /></p> 
<p>In this post, we set up a Lambda function in Account A to process DynamoDB stream events from a table in Account B. We create the Lambda execution role first, then set up the DynamoDB table with streams, configure the cross-account permissions, and finally connect them with an event source mapping.</p> 
<p>The following commands use variables to make the setup reusable and reduce manual copying of Amazon Resource Names (ARNs) between steps.</p> 
<h2>Prerequisites</h2> 
<p>To deploy this solution, you must have two AWS accounts and permission to create the necessary resources. This solution uses the AWS Command Line Interface (CLI), which you can install using <a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html">these steps</a>.</p> 
<h2>Define variables</h2> 
<p>Use the following code to define the variables:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash"># AWS Region
REGION=us-east-1

# Resource names
TABLE_NAME=Orders
ROLE_NAME=CrossAccountStreamProcessor
FUNCTION_NAME=ProcessOrders</code></pre> 
</div> 
<h2>Create Lambda execution role in Account A</h2> 
<p>In Account A, create an IAM role that will serve as the Lambda execution role:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">ROLE_ARN="$(aws iam create-role \
--role-name "$ROLE_NAME" \
--assume-role-policy-document '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}' \
--query 'Role.Arn' \
--output text)"


aws iam attach-role-policy \
--role-name “$ROLE_NAME” \
--policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaDynamoDBExecutionRole

echo "Role ARN: $ROLE_ARN"</code></pre> 
</div> 
<h2>Create Lambda function in Account A</h2> 
<p>Create the Lambda function code:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">cat &gt; index.py &lt;&lt; 'EOF'
import json

def handler(event, context):
    print(json.dumps(event, indent=2))
    return {'statusCode': 200}
EOF</code></pre> 
</div> 
<p>Package the code:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">zip function.zip index.py</code></pre> 
</div> 
<p>Create the Lambda function:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws lambda create-function \
--function-name “$FUNCTION_NAME” \
--runtime python3.12 \
--role “$ROLE_ARN” \
--handler index.handler \
--zip-file fileb://function.zip \
--region “$REGION”</code></pre> 
</div> 
<h2>Create DynamoDB table with streams in Account B</h2> 
<p>In Account B, create a DynamoDB table with streams enabled:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">STREAM_ARN="$(aws dynamodb create-table \
--table-name "$TABLE_NAME" \
--attribute-definitions AttributeName=OrderId,AttributeType=S \
--key-schema AttributeName=OrderId,KeyType=HASH \
--billing-mode PAY_PER_REQUEST \
--stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
--region "$REGION" \
--query 'TableDescription.LatestStreamArn' \
--output text)"


echo "Stream ARN: $STREAM_ARN"</code></pre> 
</div> 
<h2>Attach resource-based policy to stream in Account B</h2> 
<p>In Account B, attach a resource-based policy to your DynamoDB stream.First, create the JSON policy:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">cat &gt; stream-policy.json &lt;&lt; EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountStreamAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "$ROLE_ARN"
      },
      "Action": [
        "dynamodb:DescribeStream",
        "dynamodb:GetRecords",
        "dynamodb:GetShardIterator"
      ],
      "Resource": "*"
    }
  ]
}
EOF</code></pre> 
</div> 
<p>Next, apply the policy:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb put-resource-policy \
  --resource-arn "$STREAM_ARN" \
  --policy file://stream-policy.json \
  --region "$REGION"</code></pre> 
</div> 
<h2>Create event source mapping in Account A</h2> 
<p>In Account A, create an event source mapping between your Lambda function and the cross-account stream:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws lambda create-event-source-mapping \
--function-name “$FUNCTION_NAME” \
--event-source-arn “$STREAM_ARN” \
--starting-position LATEST \
--region “$REGION”</code></pre> 
</div> 
<h2>Considerations</h2> 
<p>Keep in mind the following when deploying this solution:</p> 
<ul> 
 <li>Both the DynamoDB table and Lambda function must be in the same AWS Region</li> 
 <li>Standard <a href="https://aws.amazon.com/dynamodb/pricing/" rel="noopener noreferrer" target="_blank">DynamoDB Streams</a> and <a href="https://aws.amazon.com/lambda/pricing/" rel="noopener noreferrer" target="_blank">Lambda</a> pricing applies; there are no additional charges for cross-account access</li> 
 <li>The stream’s resource-based policy uses the Lambda execution role ARN as the principal for fine-grained access control</li> 
 <li>The stream’s resource-based policy supports standard IAM policy features, including conditions and policy variables</li> 
 <li>Make sure to include the following required actions in your policy: <code>dynamodb:DescribeStream</code>, <code>dynamodb:GetRecords</code>, and <code>dynamodb:GetShardIterator</code></li> 
 <li>This feature works with both new and existing DynamoDB tables with streams enabled</li> 
 <li>This feature does not support <a href="https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#aws-managed-key">Amazon managed keys</a></li> 
</ul> 
<h2>Cleaning up</h2> 
<p>To avoid incurring future charges, remove the resources created in this walkthrough:</p> 
<ol> 
 <li>In Account A, delete the resources you created: 
  <div class="hide-language"> 
   <pre><code class="lang-bash">aws lambda list-event-source-mappings \
--function-name “$ProcessOrders” \
--region “$REGION”

aws lambda delete-event-source-mapping \
--uuid &lt;UUID-from-previous-command&gt; \
--region “$REGION”</code></pre> 
  </div> <p> Delete the Lambda function:</p> 
  <div class="hide-language"> 
   <pre><code class="lang-bash">aws lambda delete-function \
--function-name “$ProcessOrders” \
--region “$REGION”</code></pre> 
  </div> <p> Delete the IAM role:</p> 
  <div class="hide-language"> 
   <pre><code class="lang-bash">aws iam detach-role-policy \
--role-name “$ROLE_NAME” \
--policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaDynamoDBExecutionRole

aws iam delete-role \
--role-name “$ROLE_NAME”</code></pre> 
  </div> </li> 
 <li>In Account B, delete the DynamoDB table: 
  <div class="hide-language"> 
   <pre><code class="lang-bash">aws dynamodb delete-table \
--table-name “$TABLE_NAME”</code></pre> 
  </div> </li> 
</ol> 
<h2>Conclusion</h2> 
<p>Resource-based policy support for DynamoDB streams gives you a powerful new way to build event-driven systems across AWS account boundaries. With this feature, you can create secure, scalable pipelines without writing custom integration logic. This solution can help if you’re running a SaaS offering, consolidating logs, or processing change data centrally.</p> 
<p>DynamoDB streams with Lambda provides a managed, reliable path for real-time stream processing. Start building with <a href="https://docs.aws.amazon.com/lambda/latest/dg/services-dynamodb-eventsourcemapping.html#services-dynamodb-eventsourcemapping-cross-account" rel="noopener noreferrer" target="_blank">cross-account Lambda and DynamoDB streams</a> today and simplify your event-driven architecture.</p> 
<hr /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="author name" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/29/5323-image-2.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Lee Hannigan</h3> 
  <p><a href="https://www.linkedin.com/in/lee-hannigan/" rel="noopener" target="_blank">Lee</a> is a Sr. Amazon DynamoDB Database Engineer based in Donegal, Ireland. He brings a wealth of expertise in distributed systems, with a strong foundation in big data and analytics technologies. In his role, Lee focuses on advancing the performance, scalability, and reliability of DynamoDB while helping customers and internal teams make the most of its capabilities.</p>
 </div> 
</footer>
