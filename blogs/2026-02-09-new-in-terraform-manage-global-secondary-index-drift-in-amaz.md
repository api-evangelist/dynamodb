---
title: "New in Terraform: Manage global secondary index drift in Amazon DynamoDB"
url: "https://aws.amazon.com/blogs/database/new-in-terraform-manage-global-secondary-index-drift-in-amazon-dynamodb/"
date: "Mon, 09 Feb 2026 19:37:43 +0000"
author: "Vaibhav Bhardwaj"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
<p>If you’ve ever adjusted <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> global secondary index capacity (GSI) outside Terraform, you know how Terraform detects drift and forces unwanted reverts. With Terraform’s new <em>aws_dynamodb_global_secondary_index</em> resource, you can address this problem.</p> 
<p>The new <code>aws_dynamodb_global_secondary_index</code> resource treats each GSI as an independent resource with its own lifecycle management. You can use this feature to make capacity adjustments for GSI and tables outside of Terraform.</p> 
<p>In this post, I demonstrate how to use Terraform’s new <code>aws_dynamodb_global_secondary_index</code> resource to manage GSI drift selectively. I walk you through the limitations of current approaches and guide you through implementing the solution.</p> 
<h2>The problem: Terraform drift and GSI management</h2> 
<p>Before diving into the solution, let’s establish what <em>drift</em> means in infrastructure management. In infrastructure as code (IaC), drift occurs when the actual state of your infrastructure differs from what’s defined in your Terraform configuration. Terraform detects drift by comparing the desired state (your <code>.tf</code> configuration files), the last known state (stored in <code>terraform.tfstate</code>), and the actual state (queried from AWS). When these don’t match, Terraform reports drift and proposes changes to reconcile the difference.</p> 
<p>DynamoDB GSIs often require capacity adjustments for various operational reasons: load testing, capacity planning, emergency performance requirements, or managing warm throughput. Your DynamoDB capacity can also be changed by autoscaling events. Whenever you make these changes outside of Terraform, it creates drift between Terraform’s configuration and AWS reality.</p> 
<p>For example, let’s assume your analytics team runs a daily report that queries a GSI heavily. The report runs at 2:00 AM and needs 50 read capacity units (RCUs), but during normal hours, 5 RCUs is sufficient. Your operations team manually increases capacity before the report runs to handle the load.</p> 
<p>At 1:50 AM, your ops team increases capacity from 5 to 50 using <a href="https://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface</a> (AWS CLI). The report runs from 2:00 AM to 3:00 AM with the higher capacity. Later that day, when you run <code>terraform plan</code> to deploy an unrelated change, Terraform detects drift because the actual capacity (50) doesn’t match your configuration (5). Terraform wants to revert the capacity back to 5, which would interfere with your operational capacity management.</p> 
<h3>The common workaround and its limitations</h3> 
<p>A common workaround is to use <code>ignore_changes = [global_secondary_index]</code> in your table’s lifecycle block. This prevents Terraform from detecting capacity drift. However, this approach is too broad—it ignores all GSI changes, not just capacity. Because <code>global_secondary_index</code> is a complex nested type, <code>ignore_changes</code> only works at the top-level, not at individual attributes. If someone accidentally deletes a GSI or modifies its key schema, Terraform won’t detect it. You can’t distinguish between intentional capacity tuning and accidental GSI deletions.</p> 
<h2>The solution: Separate GSI resources</h2> 
<p>The new <code>aws_dynamodb_global_secondary_index</code> resource treats each GSI as an independent resource with its own lifecycle management. This gives you granular control over which attributes to ignore for each GSI while still detecting important changes like deletions or schema modifications.</p> 
<h3>Prerequisites</h3> 
<p>Before you begin, verify you have:</p> 
<ul> 
 <li>An AWS account with permissions to create and manage DynamoDB tables (needed to create the test resources that you will use in examples)</li> 
 <li>An <a href="https://aws.amazon.com/ec2/" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a> instance running <a href="https://aws.amazon.com/linux/" rel="noopener noreferrer" target="_blank">Amazon Linux</a> with an <a href="https://aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> role that has DynamoDB permissions</li> 
 <li><a href="https://aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a> installed and configured</li> 
 <li>AWS Terraform Provider version 6.28.0 or later (the <code>aws_dynamodb_global_secondary_index</code> resource was introduced in v6.28.0)</li> 
</ul> 
<p>The <code>aws_dynamodb_global_secondary_index</code> resource is currently marked as <em>experimental</em> in the <a href="https://registry.terraform.io/providers/hashicorp/aws/latest/docs" rel="noopener noreferrer" target="_blank">Terraform AWS provider</a>. This means the schema or behavior might change without notice, and isn’t subject to the backwards compatibility guarantee of the provider.</p> 
<p>You must set the environment variable <code>TF_AWS_EXPERIMENT_dynamodb_global_secondary_index</code> to enable this experimental resource. Without this environment variable, Terraform will return an error when attempting to use <code>aws_dynamodb_global_secondary_index</code>. Set it before running any Terraform commands:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">export TF_AWS_EXPERIMENT_dynamodb_global_secondary_index=1</code></pre> 
</div> 
<p>Test thoroughly in non-production environments before using in production. You are welcome to provide feedback at <a href="https://github.com/hashicorp/terraform-provider-aws/issues/45640" rel="noopener noreferrer" target="_blank">GitHub Issue #45640</a>.</p> 
<p>If you’re upgrading from AWS Provider v5.x to v6.x, review the <a href="https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/version-6-upgrade" rel="noopener noreferrer" target="_blank">v6.0.0 upgrade guide</a> for breaking changes before proceeding.</p> 
<p><strong>Install Terraform on Amazon Linux:</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-bash"># Update system
sudo yum update -y

# Install yum-config-manager
sudo yum install -y yum-utils

# Add HashiCorp repository
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo

# Install Terraform
sudo yum -y install terraform

# Verify installation
terraform --version</code></pre> 
</div> 
<h3>Using the new resource</h3> 
<p>Create a provisioned capacity table with two GSIs using the new separate resource method. You’ll create <code>main.tf</code> where the table and GSIs are defined as independent resources.</p> 
<p>Table and GSI keys:</p> 
<table border="1px" cellpadding="10px" class="styled-table" width="100%"> 
 <thead> 
  <tr> 
   <th>Resource</th> 
   <th>Hash key</th> 
   <th>Range key</th> 
   <th>Capacity</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>Table</td> 
   <td>id</td> 
   <td>timestamp</td> 
   <td>5/5</td> 
  </tr> 
  <tr> 
   <td>StatusUserIndex</td> 
   <td>status</td> 
   <td>user_id</td> 
   <td>5/5</td> 
  </tr> 
  <tr> 
   <td>TimestampIndex</td> 
   <td>timestamp</td> 
   <td>–</td> 
   <td>3/3</td> 
  </tr> 
 </tbody> 
</table> 
<p>Configuration:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~&gt; 6.28"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# DynamoDB Table without GSI blocks (GSIs managed separately)
# Only define attributes that are used as table keys (hash_key/range_key)
# GSI attributes are defined in the separate aws_dynamodb_global_secondary_index resources
resource "aws_dynamodb_table" "test_table" {
  name           = "GSITestTable"
  billing_mode   = "PROVISIONED"
  read_capacity  = 5
  write_capacity = 5
  hash_key       = "id"
  range_key      = "timestamp"

  attribute {
    name = "id"
    type = "S"
  }

  attribute {
    name = "timestamp"
    type = "N"
  }

  tags = {
    Name        = "GSITestTable"
    Environment = "test"
    Purpose     = "Testing new GSI resource"
  }
}

# GSI as a separate resource
resource "aws_dynamodb_global_secondary_index" "status_index" {
  table_name = aws_dynamodb_table.test_table.name
  index_name = "StatusUserIndex"

  # Provisioned throughput configuration
  provisioned_throughput {
    read_capacity_units  = 5
    write_capacity_units = 5
  }

  # key_schema now includes attribute_type (required in new resource)
  key_schema {
    attribute_name = "status"
    attribute_type = "S"
    key_type       = "HASH"
  }

  key_schema {
    attribute_name = "user_id"
    attribute_type = "S"
    key_type       = "RANGE"
  }

  # Projection configuration
  projection {
    projection_type = "ALL"
  }

  # With the new separate resource, you can now ignore specific attributes per GSI
  lifecycle {
    ignore_changes = [provisioned_throughput]
  }
}

# Second GSI to test multiple independent GSIs
resource "aws_dynamodb_global_secondary_index" "timestamp_index" {
  table_name = aws_dynamodb_table.test_table.name
  index_name = "TimestampIndex"

  # Provisioned throughput configuration
  provisioned_throughput {
    read_capacity_units  = 3
    write_capacity_units = 3
  }

  key_schema {
    attribute_name = "timestamp"
    attribute_type = "N"
    key_type       = "HASH"
  }

  # Projection configuration
  projection {
    projection_type = "KEYS_ONLY"
  }

  # This GSI is fully managed by Terraform (no ignore_changes)
}</code></pre> 
</div> 
<p>Deploy the resources:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash"># Set the required environment variable
export TF_AWS_EXPERIMENT_dynamodb_global_secondary_index=1

terraform init
terraform plan
terraform apply</code></pre> 
</div> 
<p>Test selective <code>ignore_changes</code> by manually changing the capacity of <code>StatusUserIndex</code> (the one with <code>ignore_changes</code>):</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb update-table \
  --table-name GSITestTable \
  --region us-east-1 \
  --global-secondary-index-updates '[{
    "Update": {
      "IndexName": "StatusUserIndex",
      "ProvisionedThroughput": {
        "ReadCapacityUnits": 10,
        "WriteCapacityUnits": 10
      }
    }
  }]'

# Wait for update to complete
sleep 30</code></pre> 
</div> 
<p>Running <code>terraform plan</code> shows <code>No changes</code> even though <code>StatusUserIndex</code> capacity changed to 10/10 in AWS. This occurs because of <code>ignore_changes = [provisioned_throughput]</code>.</p> 
<p>Verify drift detection still works by manually changing <code>TimestampIndex</code> (the one without <code>ignore_changes</code>):</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb update-table \
  --table-name GSITestTable \
  --region us-east-1 \
  --global-secondary-index-updates '[{
    "Update": {
      "IndexName": "TimestampIndex",
      "ProvisionedThroughput": {
        "ReadCapacityUnits": 8,
        "WriteCapacityUnits": 8
      }
    }
  }]'

# Wait for update to complete
sleep 30</code></pre> 
</div> 
<p>Running <code>terraform plan</code> detects the drift and proposes to change <code>TimestampIndex</code> capacity from 8 back to 3. This demonstrates that:</p> 
<ul> 
 <li><code>StatusUserIndex</code> shows no changes (capacity ignored as intended)</li> 
 <li><code>TimestampIndex</code> shows drift detection (capacity changes detected)</li> 
 <li>Each GSI has independent lifecycle management</li> 
 <li>You can selectively ignore specific attributes per GSI</li> 
 <li>Terraform still detects important changes on GSIs without <code>ignore_changes</code>.</li> 
</ul> 
<p>The key differences from the traditional method are that the table defines attributes used by the table itself (<code>id</code>, <code>timestamp</code>), while GSI-specific attributes (<code>status</code>, <code>user_id</code>) are defined in the separate GSI resource’s <code>key_schema</code> blocks with their <code>attribute_type</code> (required in new resource). If a GSI reuses a table attribute, that attribute remains in the table’s attribute block. The GSI is a separate resource with its own lifecycle.</p> 
<h3>Benefits of the new resource</h3> 
<p>The new resource model provides several advantages. You can now ignore specific attributes of a GSI without affecting other GSIs and automated scripts can adjust capacity based on traffic patterns without creating Terraform drift. You still track important changes like key schema modifications, confirming no accidental GSI deletions or reconfigurations. Terraform state remains the source of truth for GSI structure, while DynamoDB APIs show actual runtime capacity.</p> 
<p>Each GSI can have its own lifecycle rules, providing independent management. The new resource model follows Terraform best practices where each resource manages one logical infrastructure component, dependencies are explicit through resource references, and state management is more straightforward.</p> 
<p>The new resource fully supports <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/warm-throughput.html" rel="noopener noreferrer" target="_blank">warm throughput configuration for on-demand tables</a>. Warm throughput is a DynamoDB capability that you can use to specify baseline capacity for on-demand tables, helping you manage performance and costs more predictably. This is how you can test it.</p> 
<p>Create <code>ondemand.tf</code>:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">resource "aws_dynamodb_table" "ondemand_test" {
  name         = "OnDemandGSITest"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"

  attribute {
    name = "id"
    type = "S"
  }
}

resource "aws_dynamodb_global_secondary_index" "category_index" {
  table_name = aws_dynamodb_table.ondemand_test.name
  index_name = "CategoryIndex"

  key_schema {
    attribute_name = "category"
    attribute_type = "S"
    key_type       = "HASH"
  }

  # Projection configuration
  projection {
    projection_type = "ALL"
  }

  # Warm throughput configuration (attribute, not a block)
  warm_throughput = {
    read_units_per_second  = 13000
    write_units_per_second = 5000
  }

  lifecycle {
    # Allow manual warm throughput tuning
    ignore_changes = [warm_throughput]
  }
}</code></pre> 
</div> 
<p>Deploy and test:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash"># Set the required environment variable if not already set
export TF_AWS_EXPERIMENT_dynamodb_global_secondary_index=1

terraform apply

# Change warm throughput manually
aws dynamodb update-table \
  --table-name OnDemandGSITest \
  --region us-east-1 \
  --global-secondary-index-updates '[{
    "Update": {
      "IndexName": "CategoryIndex",
      "WarmThroughput": {
        "ReadUnitsPerSecond": 14000,
        "WriteUnitsPerSecond": 5100
      }
    }
  }]'

# Wait for update
sleep 30

# Run terraform plan
terraform plan</code></pre> 
</div> 
<p>Terraform shows <code>No changes</code> because warm throughput changes are ignored as expected.</p> 
<p>Before moving to the next section, destroy the on-demand test resources:<code>terraform destroy</code></p> 
<h3>Migration example</h3> 
<p>Now that you’ve seen how the new resource works, let’s walk through a complete hands-on migration of existing infrastructure. Start with a table using the traditional nested GSI approach, then migrate it to the new separate resource method without any downtime.</p> 
<h4>Step 1: Create infrastructure with traditional method</h4> 
<p>Create a DynamoDB table with a GSI using the traditional nested block approach.</p> 
<p>Create a file called <code>migration-old.tf</code>:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~&gt; 6.28"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Traditional approach: GSI defined as nested block
resource "aws_dynamodb_table" "products" {
  name           = "ProductsTable"
  billing_mode   = "PROVISIONED"
  read_capacity  = 5
  write_capacity = 5
  hash_key       = "ProductId"

  attribute {
    name = "ProductId"
    type = "S"
  }

  attribute {
    name = "Category"
    type = "S"
  }

  # GSI defined as nested block (TRADITIONAL METHOD)
  global_secondary_index {
    name               = "CategoryIndex"
    hash_key           = "Category"
    projection_type    = "ALL"
    read_capacity      = 3
    write_capacity     = 3
  }

  tags = {
    Name        = "ProductsTable"
    Environment = "migration-demo"
  }
}</code></pre> 
</div> 
<p>Deploy this infrastructure:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">terraform init
terraform plan
terraform apply</code></pre> 
</div> 
<p>Verify the table and GSI were created:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb describe-table --table-name ProductsTable --region us-east-1 \
  --query 'Table.GlobalSecondaryIndexes[0].IndexName'</code></pre> 
</div> 
<p>Output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">CategoryIndex</code></pre> 
</div> 
<h4>Step 2: Prepare for migration</h4> 
<p>Before migrating, backup your Terraform state:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform state pull &gt; backup-before-migration.tfstate</code></pre> 
</div> 
<p>Set the required environment variable:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">export TF_AWS_EXPERIMENT_dynamodb_global_secondary_index=1</code></pre> 
</div> 
<h4>Step 3: Update your Terraform configuration</h4> 
<p>Create a new file called <code>migration-new.tf</code> with the updated configuration. Keep both files for now—you will remove the old one after import.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~&gt; 6.28"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Updated table: GSI block removed
resource "aws_dynamodb_table" "products" {
  name           = "ProductsTable"
  billing_mode   = "PROVISIONED"
  read_capacity  = 5
  write_capacity = 5
  hash_key       = "ProductId"

  # Only define attributes used by the table's own keys
  attribute {
    name = "ProductId"
    type = "S"
  }

  tags = {
    Name        = "ProductsTable"
    Environment = "migration-demo"
  }
}

# NEW: GSI as a separate resource
resource "aws_dynamodb_global_secondary_index" "category_index" {
  table_name = aws_dynamodb_table.products.name
  index_name = "CategoryIndex"

  # Provisioned throughput configuration
  provisioned_throughput {
    read_capacity_units  = 3
    write_capacity_units = 3
  }

  key_schema {
    attribute_name = "Category"
    attribute_type = "S"
    key_type       = "HASH"
  }

  # Projection configuration
  projection {
    projection_type = "ALL"
  }

  # Allow ops team to adjust capacity without Terraform reverting it
  lifecycle {
    ignore_changes = [provisioned_throughput]
  }
}</code></pre> 
</div> 
<h4>Step 4: Remove the old configuration</h4> 
<p>Now remove or rename the old file:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">mv migration-old.tf migration-old.tf.backup</code></pre> 
</div> 
<p>At this point, if you run <code>terraform plan</code>, you’ll see that Terraform wants to remove the GSI from the table (because the nested block is gone) and create a new separate GSI resource.</p> 
<p><em>Don’t apply yet.</em> This would cause downtime. Instead, import the existing GSI.</p> 
<h4>Step 5: Import the existing GSI</h4> 
<p>Import the existing GSI into the new resource’s state:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash"># Import format: 'table_name,index_name'
terraform import aws_dynamodb_global_secondary_index.category_index \
  'ProductsTable,CategoryIndex'</code></pre> 
</div> 
<p>Output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws_dynamodb_global_secondary_index.category_index: Importing from ID "ProductsTable,CategoryIndex"...
aws_dynamodb_global_secondary_index.category_index: Import prepared!
  Prepared aws_dynamodb_global_secondary_index for import
aws_dynamodb_global_secondary_index.category_index: Refreshing state... [id=ProductsTable,CategoryIndex]

Import successful!</code></pre> 
</div> 
<h4>Step 6: Verify the migration</h4> 
<p>Run <code>terraform plan</code> to verify:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform plan</code></pre> 
</div> 
<p>Expected output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws_dynamodb_table.products: Refreshing state... [id=ProductsTable]
aws_dynamodb_global_secondary_index.category_index: Refreshing state... [id=ProductsTable,CategoryIndex]

No changes. Your infrastructure matches the configuration.</code></pre> 
</div> 
<p>If you see <code>No changes</code>, the migration was successful. The GSI is now managed as a separate resource.</p> 
<h3>Migration summary</h3> 
<p>To complete a migration, you started with a traditional nested GSI configuration, which you then migrated to separate GSI resources without downtime using <code>terraform import</code>. You then verified the migration with <code>terraform plan</code> showing <code>No changes</code>, after which you successfully transitioned to the new resource model.</p> 
<p><strong>Key takeaways:</strong></p> 
<ul> 
 <li>Migration uses <code>terraform import</code></li> 
 <li>No AWS resources are modified or recreated</li> 
 <li>The GSI continues to exist throughout the migration with zero downtime</li> 
 <li>After migration, you have granular control over what to ignore with <code>ignore_changes</code></li> 
 <li>The migration process is safe and reversible</li> 
</ul> 
<h3>Migration considerations</h3> 
<p>Do not combine <code>aws_dynamodb_global_secondary_index</code> resources with <code>global_secondary_index</code> blocks on <code>aws_dynamodb_table</code>. Doing so might cause conflicts, perpetual differences, and GSIs being overwritten.</p> 
<p>When migrating, follow these steps:</p> 
<ol> 
 <li><strong>Backup state</strong>: <code>terraform state pull &gt; backup.tfstate</code></li> 
 <li><strong>Set environment variable</strong>: <code>export TF_AWS_EXPERIMENT_dynamodb_global_secondary_index=1</code></li> 
 <li><strong>Update configuration</strong>: Remove the GSI block from table the table and create a new GSI resource</li> 
 <li><strong>Import existing GSI</strong>: <code>terraform import &lt;resource&gt; 'table_name,index_name'</code></li> 
 <li><strong>Verify</strong>: Run <code>terraform plan</code>, it should show <code>No changes</code></li> 
 <li><strong>Test</strong>: Manually change capacity and verify that Terraform ignores the change</li> 
</ol> 
<p>You won’t experience downtime during migration if done correctly using <code>terraform import</code>. The GSI continues to exist in AWS throughout the migration. The <code>terraform import</code> command only updates Terraform’s state file—it doesn’t modify AWS resources.</p> 
<p>If your table has multiple GSIs, migrate them one at a time:</p> 
<ol> 
 <li>Import the first GSI and verify with <code>terraform plan</code></li> 
 <li>Import the second GSI and verify with <code>terraform plan</code></li> 
 <li>Continue until all GSIs are migrated</li> 
</ol> 
<p>This reduces risk and simplifies troubleshooting.</p> 
<h2>Comparison: Traditional compared to new method</h2> 
<p>The following table summarizes the key differences between the traditional nested block approach and the new separate resource method:</p> 
<table border="1px" cellpadding="10px" class="styled-table" width="100%"> 
 <thead> 
  <tr> 
   <th>Aspect</th> 
   <th>Traditional method (nested block)</th> 
   <th>New method (separate resource)</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>Resource enablement</td> 
   <td>No environment variable needed</td> 
   <td>Requires <code>TF_AWS_EXPERIMENT_dynamodb_global_secondary_index=1</code></td> 
  </tr> 
  <tr> 
   <td>Granular <code>ignore_changes</code></td> 
   <td>Not supported</td> 
   <td>Supported</td> 
  </tr> 
  <tr> 
   <td>Independent GSI management</td> 
   <td>All GSIs managed together</td> 
   <td>Each GSI managed independently</td> 
  </tr> 
  <tr> 
   <td>Drift detection</td> 
   <td>All-or-nothing</td> 
   <td>Selective per GSI</td> 
  </tr> 
  <tr> 
   <td>Lifecycle rules</td> 
   <td>Applies to all GSIs</td> 
   <td>Per-GSI lifecycle rules</td> 
  </tr> 
  <tr> 
   <td>State management</td> 
   <td>Complex nested state</td> 
   <td>Straightforward flat state</td> 
  </tr> 
  <tr> 
   <td>Capacity configuration</td> 
   <td>Top-level attributes (<code>read_capacity</code>, <code>write_capacity</code>)</td> 
   <td>Block syntax (<code>provisioned_throughput</code> block)</td> 
  </tr> 
  <tr> 
   <td>Projection configuration</td> 
   <td>Top-level attribute (<code>projection_type</code>)</td> 
   <td>Block syntax (<code>projection</code> block)</td> 
  </tr> 
  <tr> 
   <td>Warm throughput support</td> 
   <td>Limited</td> 
   <td>Full support (attribute syntax: <code>warm_throughput = { }</code>)</td> 
  </tr> 
  <tr> 
   <td>Migration complexity</td> 
   <td>N/A</td> 
   <td>Requires import process</td> 
  </tr> 
  <tr> 
   <td>Backward compatibility</td> 
   <td>Existing method</td> 
   <td>Cannot mix with traditional method</td> 
  </tr> 
  <tr> 
   <td>Stability</td> 
   <td>Stable</td> 
   <td>Experimental (schema might change)</td> 
  </tr> 
 </tbody> 
</table> 
<h2>Clean up</h2> 
<p>To avoid incurring future charges, delete the resources you created in this walkthrough:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code"># Destroy all Terraform-managed resources
terraform destroy

# Confirm the deletion when prompted
# Type 'yes' to proceed</code></pre> 
</div> 
<p>If you created any resources manually during testing, make sure to delete those as well through the AWS Management Console or AWS CLI to avoid incurring future costs.</p> 
<h2>Conclusion</h2> 
<p>In this post, I showed you how the new <code>aws_dynamodb_global_secondary_index</code> resource solves the long-standing challenge of managing DynamoDB GSI drift in Terraform. The all-or-nothing nature of ignoring nested <code>global_secondary_index</code> blocks created a gap between operational flexibility and infrastructure governance.</p> 
<p>By treating GSIs as first-class resources, you gain granular control with selective <code>ignore_changes</code> for specific GSI attributes, independent management where each GSI has its own lifecycle rules, better drift detection that tracks important changes while allowing operational adjustments, and a more straightforward architecture with separation of concerns between table and index configuration.</p> 
<p>Remember that the <code>aws_dynamodb_global_secondary_index</code> resource is currently marked as <em>experimental</em>. While it provides powerful capabilities for managing GSI drift, be aware that:</p> 
<ul> 
 <li>The schema or behavior might change in future provider versions</li> 
 <li>You must set the environment variable <code>TF_AWS_EXPERIMENT_dynamodb_global_secondary_index=1</code> to enable this resource</li> 
 <li>It’s not subject to the backwards compatibility guarantee of the provider</li> 
 <li>You can’t mix this resource with traditional <code>global_secondary_index</code> blocks on the same table</li> 
</ul> 
<p>Always test thoroughly in non-production environments and monitor provider release notes for updates. If you have feedback, provide it at <a href="https://github.com/hashicorp/terraform-provider-aws/issues/45640" rel="noopener noreferrer" target="_blank">GitHub Issue #45640</a> to help shape the future of this feature.</p> 
<hr /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Vaibhav Bhardwaj" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/02/06/vb.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Vaibhav Bhardwaj</h3> 
  <p><a href="https://www.linkedin.com/in/vbhardwaj7/" rel="noopener" target="_blank">Vaibhav</a> is a Senior DynamoDB Specialist Solutions Architect based at AWS Singapore. He’s a serverless enthusiast with 19 years of experience and likes working with customers to design architectures for applications that demand high performance, scalability, and reliability with DynamoDB.</p>
 </div> 
</footer>
