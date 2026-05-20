---
title: "Efficiently compare items across two Amazon DynamoDB tables"
url: "https://aws.amazon.com/blogs/database/efficiently-compare-items-across-two-amazon-dynamodb-tables/"
date: "Mon, 17 Nov 2025 21:30:34 +0000"
author: "Jason Hunter"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
In this post, we show an algorithm to efficiently compare two Amazon DynamoDB tables and find the differences between their items. We provide an example where two tables, each containing approximately half a billion items, are compared in less than 7 minutes, for less than $10.
