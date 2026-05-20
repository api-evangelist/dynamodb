---
title: "Rate-limiting calls to Amazon DynamoDB using Python Boto3, Part 2: Distributed Coordination"
url: "https://aws.amazon.com/blogs/database/rate-limiting-calls-to-amazon-dynamodb-using-python-boto3-part-2-distributed-coordination/"
date: "Thu, 13 Nov 2025 17:57:34 +0000"
author: "Jason Hunter"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
Part 1 of this series showed how to rate-limit calls to Amazon DynamoDB by using Python Boto3 event hooks. In this post, I expand on the concept and show how to rate-limit calls in a distributed environment, where you want a maximum allowed rate across the full set of clients but can’t use direct client-to-client communication.
