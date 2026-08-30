---
title: "How Channel Corporation modernized their architecture with Amazon DynamoDB, Part 3: User and Badge"
url: "https://aws.amazon.com/blogs/database/how-channel-corporation-modernized-their-architecture-with-amazon-dynamodb-part-3-user-and-badge/"
date: "2026-08-25"
author: "Haibin (Binu) Lee"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
Channel Corporation shares how they split their all-purpose Amazon DynamoDB User table into role-specific tables, moving Badge data into a dedicated UserBadge table to stop transaction-conflict throttling and GSI back pressure, and how they ran a zero-downtime online migration using DynamoDB Export and Import with Amazon S3 and AWS Glue.
