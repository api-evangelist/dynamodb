---
title: "Best practices for Amazon DynamoDB Global Tables – Part 3: Validating regional resilience with AWS Fault Injection Service"
url: "https://aws.amazon.com/blogs/database/best-practices-for-amazon-dynamodb-global-tables-part-3-validating-regional-resilience-with-aws-fault-injection-service/"
date: "2026-05-20"
author: "Lee Hannigan"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
In this post, we show you how to use AWS Fault Injection Service (AWS FIS) to validate that your application handles regional disruptions the way you expect, by running controlled experiments against your DynamoDB global tables. We cover both multi-Region strong consistency (MRSC) and multi-Region eventually consistent (MREC) global tables, because AWS FIS works differently with each.
