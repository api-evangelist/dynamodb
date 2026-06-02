---
title: "Introducing ExtendDB: An open source DynamoDB-compatible adapter with pluggable storage backends"
url: "https://aws.amazon.com/blogs/database/introducing-extenddb-an-open-source-dynamodb-compatible-adapter-with-pluggable-storage-backends/"
date: "2026-05-20"
author: "Lee Hannigan"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
Today, we are announcing ExtendDB, an open source Amazon DynamoDB-compatible adapter with pluggable storage backends, released under the Apache 2.0 License. ExtendDB implements the DynamoDB wire protocol and ships with PostgreSQL as its first backend, so any AWS SDK, CLI, or tool that works with DynamoDB works with ExtendDB unchanged. In this post, we introduce ExtendDB, walk through getting started, and explain the architecture.
