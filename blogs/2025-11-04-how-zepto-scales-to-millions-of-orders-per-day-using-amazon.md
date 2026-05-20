---
title: "How Zepto scales to millions of orders per day using Amazon DynamoDB"
url: "https://aws.amazon.com/blogs/database/how-zepto-scales-to-millions-of-orders-per-day-using-amazon-dynamodb/"
date: "Tue, 04 Nov 2025 22:59:48 +0000"
author: "Rahul Bennapalan"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
In this post, we describe how Zepto transformed its data infrastructure from a centralized relational database to a distributed system for select use cases. We discuss the challenges encountered with Zepto’s original architecture to support the business scale, the shift towards using key-value storage for cases where eventual consistency was acceptable, and Zepto’s adoption of Amazon DynamoDB.
