---
title: "Multi-key support for Global Secondary Index in Amazon DynamoDB"
url: "https://aws.amazon.com/blogs/database/multi-key-support-for-global-secondary-index-in-amazon-dynamodb/"
date: "Thu, 20 Nov 2025 18:16:34 +0000"
author: "Esteban Serna Parra"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
Amazon DynamoDB has announced support for up to 8 attributes in composite keys for Global Secondary Indexes (GSIs). Now, you can specify up to four partition keys and four sort keys to identify items as part of a GSI, allowing you to query data at scale across multiple dimensions. In this post we show you how to design similar data models more efficiently using Global Secondary Indexes with the additional attribute support in composite keys and provide examples of DynamoDB data models with reduced complexity.
