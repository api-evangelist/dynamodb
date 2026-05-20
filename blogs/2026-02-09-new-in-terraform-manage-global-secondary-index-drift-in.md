---
title: "New in Terraform: Manage global secondary index drift in Amazon DynamoDB"
url: "https://aws.amazon.com/blogs/database/new-in-terraform-manage-global-secondary-index-drift-in-amazon-dynamodb/"
date: "Mon, 09 Feb 2026 19:37:43 +0000"
author: "Vaibhav Bhardwaj"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
The new aws_dynamodb_global_secondary_index resource treats each GSI as an independent resource with its own lifecycle management. You can use this feature to make capacity adjustments for GSI and tables outside of Terraform. In this post, I demonstrate how to use Terraform’s new aws_dynamodb_global_secondary_index resource to manage GSI drift selectively.
