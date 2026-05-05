---
title: "Multi-key support for Global Secondary Index in Amazon DynamoDB"
url: "https://aws.amazon.com/blogs/database/multi-key-support-for-global-secondary-index-in-amazon-dynamodb/"
date: "Thu, 20 Nov 2025 18:16:34 +0000"
author: "Esteban Serna Parra"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
<p><a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> has announced support for up to <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html#GSI.MultiAttributeKeys" rel="noopener noreferrer" target="_blank">8 attributes in composite keys for Global Secondary Indexes</a> (GSIs). Now, you can specify up to four partition keys and four sort keys to identify items as part of a GSI, allowing you to query data at scale across multiple dimensions.</p> 
<p>Amazon DynamoDB is a serverless, fully managed, distributed NoSQL database with single-digit millisecond performance at any scale. A key feature of DynamoDB is its <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html#HowItWorks.CoreComponents.PrimaryKey" rel="noopener noreferrer" target="_blank">schema flexibility</a>. Apart from the <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html#HowItWorks.CoreComponents.PrimaryKey" rel="noopener noreferrer" target="_blank">primary key</a>, everything else is optional. You can start with a simple table that has a partition key (PK) and, optionally, a sort key (SK), both as string attributes, and you are ready to build your data model.</p> 
<p>Sometimes applications need to use more than one attribute in the partition key to increase cardinality. Similarly, with the sort key when writing and querying data, to provide additional flexibility in how data is filtered. For example, if an access pattern requires retrieving transactions by status and date, or sensor readings filtered by both reading type and timestamp, using two attributes in the sort key greatly increases application efficiency. This is where you had to get creative with your solutions, <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-sort-keys.html" rel="noopener noreferrer" target="_blank">concatenating attributes</a> together at the application side to create a composite of two or more attributes.</p> 
<p>In this post we show you how to design similar data models more efficiently using Global Secondary Indexes with the additional attribute support in composite keys and provide examples of DynamoDB data models with reduced complexity.</p> 
<h2>Understanding partition and sort keys</h2> 
<p>The partition key represents the known element in your queries, it is basically the <code>user_id</code>, <code>account_id</code>, <code>sensor_id</code>, <code>player_id</code>, and more; the identifier you need to use while retrieving data. Without this information, finding items in your table is less efficient. The sort key provides answers to questions about items that share the same partition key. For example, “<em>Find transactions by date for </em><strong><em>this account</em></strong>.” or “<em>Show sensor readings and alarms for </em><strong><em>this device</em></strong><em> over time?</em>” or “<em>What items are in </em><strong><em>this customer’s shopping cart</em></strong><em>?</em>”</p> 
<p>With the new GSIs support for up to four partition keys and four sort keys, your approach to multi-dimensional queries can change. By using this capability, developer workarounds are no longer needed, maintaining the performance of DynamoDB and further increasing its flexibility.</p> 
<h3>Core concepts</h3> 
<p>DynamoDB tables are built around two concepts:</p> 
<ul> 
 <li>The <strong>partition key is the main identifier, which dictates where your data is</strong> stored. This represents the “known element” in your access pattern. The partition key must be provided in any API operation that is efficient, such as the <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.html" rel="noopener noreferrer" target="_blank">query API</a>.</li> 
 <li>The <strong>sort key</strong> is an optional attribute for you to store related items together and query them efficiently using range operations. Using sort keys makes one-to-many relationships possible and generates <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithItemCollections.html" rel="noopener noreferrer" target="_blank">item collections</a> (items which have the same partition key).</li> 
</ul> 
<p>You can combine different entity types in the sort key, such as <em>entity type</em>, <em>status</em> and <em>timestamp</em> in ISO8601 by concatenating them using the <code>#</code> character and leverage the sort key conditions to retrieve specific queries. Let’s see this example where you will retrieve information from the customer <code>C#1A2B3C</code>.</p> 
<ul> 
 <li>To retrieve all the orders. Use the sort key condition <strong>BEGINS_WITH</strong> <code>ORDER#</code></li> 
 <li>If you only want to get the <code>PENDING</code> orders you can also use the sort key condition <strong>BEGINS_WITH</strong> including the status you are looking for <code>ORDER#PENDING</code>.</li> 
 <li>Finally, if you want to retrieve all the pending orders in between two dates, use the sort key condition <strong>BETWEEN</strong> <code>ORDER#PENDING#2025-11-01</code> <strong>AND</strong> <code>ORDER#PENDING#2025-11-04</code>. Notice how every time we provide more details to retrieve a smaller subset of data.</li> 
</ul> 
<p>When designing your data model, the partition key answers, <em>“What am I looking for?”</em> or <em>“What is the main thing”</em> in other words the <strong>must-know</strong> information to locate the data. The sort key answers, <em>“What aspects of the thing I want?”</em> or <em>“What details helps me to narrow it down?”</em> in other words <strong>how do I organize and filter the data?</strong></p> 
<p>For example: For the customer 1A2B3C, which items are active in the shopping cart?</p> 
<p>The <em>must-know</em> information is the <strong><em>customer 1A2B3C</em></strong>. This is the main thing you are looking for. You can’t find any shopping cart items without knowing which customer they belong to.</p> 
<p><em>How do I organize and filter the data? </em><strong><em>active items in the shopping cart</em></strong>. You will organize the items by <code>ACTIVE</code> status in the shopping cart, adding the SKU for uniqueness.</p> 
<p>The following table shows a sample data model with three shopping cart items for the customer 1A2B3C:</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td><strong>Partition Key</strong></td> 
   <td><strong>Sort Key</strong></td> 
   <td><strong>status</strong></td> 
   <td><strong>sku</strong></td> 
   <td><strong>date</strong></td> 
  </tr> 
  <tr> 
   <td>C#1A2B3C</td> 
   <td>CART#S#ACTIVE#SKU#123ABC</td> 
   <td>ACTIVE</td> 
   <td>123ABC</td> 
   <td>2025-11-04T10:00:00</td> 
  </tr> 
  <tr> 
   <td>C#1A2B3C</td> 
   <td>CART#S#ACTIVE#SKU#234BCD</td> 
   <td>ACTIVE</td> 
   <td>234BCD</td> 
   <td>2025-11-04T10:05:00</td> 
  </tr> 
  <tr> 
   <td>C#1A2B3C</td> 
   <td>CART#S#SAVED#SKU#345CDE</td> 
   <td>SAVED</td> 
   <td>345CDE</td> 
   <td>2025-11-04T10:08:00</td> 
  </tr> 
 </tbody> 
</table> 
<p>When you need to query across multiple attributes in your sort key, the common pattern was to concatenate into a composite sort key. For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">
PK: C#1A2B3C
SK: ORDER#PENDING#2024-11-01T10:30:00Z</code></pre> 
</div> 
<h2>Introducing expanded composite keys for Global Secondary Indexes</h2> 
<p>Global Secondary Indexes support multi-attribute keys, allowing you to compose partition keys and sort keys from multiple attributes. Each attribute maintains its own data type (string, number or binary) providing flexible querying capabilities. GSIs are <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-indexes-general-sparse-indexes.html" rel="noopener noreferrer" target="_blank">sparse</a>, If any component of a composite key is missing, items won’t be indexed, like with single keys GSI.</p> 
<ul> 
 <li><strong>Multiple partition keys</strong>: Combine up to four attributes as partition keys (example: tenant, customer, department).</li> 
 <li><strong>Multiple sort keys</strong>: Define up to four sort key attributes with specific query patterns.</li> 
 <li><strong>Native data types:</strong> Each attribute keeps its type, and no string conversion and concatenation is required.</li> 
 <li><strong>Effective querying</strong>: Query with increasingly specific attribute combinations, without restructuring your data.</li> 
 <li><strong>Simple <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-sharding.html" rel="noopener noreferrer" target="_blank">sharding</a> techniques</strong>: Use two or more partition keys to reduce the risk of having <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html" rel="noopener noreferrer" target="_blank">hot partitions</a>. Solve them by implementing an intelligent sharding technique, find information in your data model to distribute your traffic.</li> 
</ul> 
<p>The following query patterns are supported:</p> 
<ul> 
 <li>Queries require equality conditions (<code>=</code>) on all partition key attributes.</li> 
 <li>Sort key conditions are optional and can use up to 4 attributes with equality (<code>=</code>) conditions.</li> 
 <li>Range conditions (<code>&lt;</code>, <code>&gt;</code>, <code>&lt;=</code>, <code>&gt;=</code>, <code>BETWEEN</code>, <code>BEGINS_WITH</code>) are only supported on the last sort key attribute.</li> 
 <li>You can’t skip sort keys in your query; for example, a query with all the partition keys but only the first and third sort keys only is not supported.</li> 
 <li>You can provide partial sort keys from left to right. For example, a query where you specify only the first sort key, or only the first and the second sort key is supported.</li> 
 <li>Maintain the same DynamoDB performance while reducing application complexity.</li> 
</ul> 
<h3><strong>How composite keys work</strong></h3> 
<p>Let’s walk through a complete example to see expanded composite keys for GSIs in action.</p> 
<h4><strong>Scenario 1: Orders dashboard</strong></h4> 
<p>You are building a system that tracks orders, it has the following requirements:</p> 
<ul> 
 <li>The system tracks the orders by order id to update its status and metadata.</li> 
 <li>Users can query their orders by status (ACTIVE, PENDING, COMPLETED) with amount thresholds, such as, ACTIVE orders on November 4<sup>th</sup> greater than $100.</li> 
</ul> 
<h4>Base table design</h4> 
<p>From the two access patterns we have in the Scenario 1, only one is not related to the user. Use the tracking system for the base table, it will allow you to scale the system as the order is the most atomic piece of information.</p> 
<p>The <em>must-know information</em> is the <strong><em>order_id</em></strong>. Without an <em>order_id</em> you will have to <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Scan.html" rel="noopener noreferrer" target="_blank">scan</a> the entire table. To help organize orders by date for the other access patterns, use a <a href="https://github.com/segmentio/ksuid" rel="noopener noreferrer" target="_blank">K-sortable unique identifier</a> (KSUID), which is sortable by generation time.</p> 
<p>Since this example doesn’t include any query on the order id, there is no need for a sort key. The following table shows a sample data model with three orders for the customer 1A2B3C:</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td><strong>Partition Key</strong></td> 
   <td></td> 
   <td></td> 
   <td></td> 
   <td></td> 
   <td></td> 
   <td></td> 
  </tr> 
  <tr> 
   <td><strong>order_id</strong></td> 
   <td><strong>customer_id</strong></td> 
   <td><strong>order_date</strong></td> 
   <td><strong>amount</strong></td> 
   <td><strong>status</strong></td> 
   <td><strong>acc_type</strong></td> 
   <td><strong>org_id</strong></td> 
  </tr> 
  <tr> 
   <td>KSUID1</td> 
   <td>C#1A2B3C</td> 
   <td>2025-11-04</td> 
   <td>200</td> 
   <td>ACTIVE</td> 
   <td>A</td> 
   <td>OMEGA</td> 
  </tr> 
  <tr> 
   <td>KSUID2</td> 
   <td>C#1A2B3C</td> 
   <td>2025-11-04</td> 
   <td>145</td> 
   <td>PENDING</td> 
   <td>A</td> 
   <td>OMEGA</td> 
  </tr> 
  <tr> 
   <td>KSUID3</td> 
   <td>C#1A2B3C</td> 
   <td>2025-11-04</td> 
   <td>110</td> 
   <td>PENDING</td> 
   <td>B</td> 
   <td>BRAVO</td> 
  </tr> 
 </tbody> 
</table> 
<div class="hide-language"> 
 <pre><code class="lang-bash">Base Table: Orders
Partition Key: order_id
Attributes: customer_id, status, order_date, amount, acc_type and org_id</code></pre> 
</div> 
<h4>Composite keys GSI design</h4> 
<p>For the dashboard queries, you create a GSI where <code>customer_id</code> is the partition key, and the sort key is composed of <code>status</code>, <code>order_date</code>, and <code>amount</code>. Since inequalities need to be the last sort key attribute, positioning <code>amount</code> last means you can use range queries to filter by price thresholds.</p> 
<p>The <em>must-know information</em> is the <strong><em>customer 1A2B3C</em></strong>. You can’t find any order without knowing which customer it belongs to. <em>How do I organize and filter the data? </em>You need to organize and filter by <strong><em>active orders for a given date that are greater than $100</em></strong>. Using a composite key using status and date, you can retrieve the orders for this customer.</p> 
<p>The following table shows a sample data model with three orders for the customer 1A2B3C:</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td><strong>GSI PK</strong></td> 
   <td colspan="3"><strong>GSI SK</strong></td> 
   <td></td> 
   <td></td> 
   <td></td> 
  </tr> 
  <tr> 
   <td><strong>customer_id</strong></td> 
   <td><strong>status</strong></td> 
   <td><strong>order_date</strong></td> 
   <td><strong>amount</strong></td> 
   <td><strong>order_id</strong></td> 
   <td><strong>acc_type</strong></td> 
   <td><strong>org_id</strong></td> 
  </tr> 
  <tr> 
   <td>C#1A2B3C</td> 
   <td>ACTIVE</td> 
   <td>2025-11-04</td> 
   <td>200</td> 
   <td>KSUID1</td> 
   <td>A</td> 
   <td>OMEGA</td> 
  </tr> 
  <tr> 
   <td>C#1A2B3C</td> 
   <td>PENDING</td> 
   <td>2025-11-04</td> 
   <td>145</td> 
   <td>KSUID2</td> 
   <td>A</td> 
   <td>OMEGA</td> 
  </tr> 
  <tr> 
   <td>C#1A2B3C</td> 
   <td>PENDING</td> 
   <td>2025-11-04</td> 
   <td>110</td> 
   <td>KSUID3</td> 
   <td>B</td> 
   <td>BRAVO</td> 
  </tr> 
 </tbody> 
</table> 
<div class="hide-language"> 
 <pre><code class="lang-bash">GSI: OrdersByStatusDateAmount
Partition Key: customer_id
Sort Key 1: status (equality condition)
Sort Key 2: order_date (equality condition)
Sort Key 3: amount (range condition)</code></pre> 
</div> 
<p>In the following AWS CLI command, you will retrieve the orders for the customer 1A2B3C.</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb query \
    --table-name orders-table \
    --index-name OrdersByStatusDateAmount \
    --key-condition-expression "customer_id = :cust" \
    --expression-attribute-values '{
        ":cust": {"S": "1A2B3C"}
    }'
</code></pre> 
</div> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
    "Items": [
        {
            "org_id": {"S": "OMEGA"},
            "order_date": {"S": "2025-11-04"},
            "status": {"S": "ACTIVE"},
            "acc_type": {"S": "A"},
            "customer_id": {"S": "1A2B3C"},
            "amount": {"N": "200"},
            "order_id": {"S": "KSUID1"}
        },
        {
            "org_id": {"S": "BRAVO"},
            "order_date": {"S": "2025-11-04"},
            "status": {"S": "PENDING"},
            "acc_type": {"S": "B"},
            "customer_id": {"S": "1A2B3C"},
            "amount": {"N": "110"},
            "order_id": {"S": "KSUID3"}
        },
        {
            "org_id": {"S": "OMEGA"},
            "order_date": {"S": "2025-11-04"},
            "status": {"S": "PENDING"},
            "acc_type": {"S": "A"},
            "customer_id": {"S": "1A2B3C"},
            "amount": {"N": "145"},
            "order_id": {"S": "KSUID2"}
        }
    ],
    "Count": 3,
    "ScannedCount": 3,
    "ConsumedCapacity": null
}
</code></pre> 
</div> 
<p>The following AWS CLI command queries the orders for the customer 1A2B3C that are in a PENDING status:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb query \
    --table-name orders-table \
    --index-name OrdersByStatusDateAmount \
    --key-condition-expression "customer_id = :cust AND #status = :status" \
    --expression-attribute-names '{"#status": "status"}' \
    --expression-attribute-values '{
        ":cust": {"S": "1A2B3C"},
        ":status": {"S": "PENDING"}
    }' 
</code></pre> 
</div> 
<p>The following AWS CLI command retrieves the orders for the customer 1A2B3C that are in a PENDING status on November 4th:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb query \
    --table-name orders-table \
    --index-name OrdersByStatusDateAmount \
    --key-condition-expression "customer_id = :cust AND #status = :status AND order_date = :date" \
    --expression-attribute-names '{"#status": "status"}' \
    --expression-attribute-values '{
        ":cust": {"S": "1A2B3C"},
        ":status": {"S": "PENDING"},
        ":date": {"S": "2025-11-04"}
    }'
</code></pre> 
</div> 
<p>The following AWS CLI command queries the orders for the customer 1A2B3C that are in a PENDING status on November 4th with an amount greater than $100.</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws dynamodb query \
    --table-name orders-table \
    --index-name OrdersByStatusDateAmount \
    --key-condition-expression "customer_id = :cust AND #status = :status AND order_date = :date AND amount &gt; :min_amount" \
    --expression-attribute-names '{"#status": "status"}' \
    --expression-attribute-values '{
        ":cust": {"S": "1A2B3C"},
        ":status": {"S": "PENDING"},
        ":date": {"S": "2025-11-04"},
        ":min_amount": {"N": "100"}
    }'
</code></pre> 
</div> 
<h3><strong>Scenario 2: Evolution – growing in traffic</strong></h3> 
<p>Now imagine the solution is a success and your largest enterprise customers will push order statuses at a rate that is higher than 500 times per second. The requirements remain the same, users need to query their order by status within some amount thresholds.</p> 
<p>With the introduction of this large enterprise customer, we are facing the possibility of <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html" rel="noopener noreferrer" target="_blank">hot partitions</a>. Updating one order status from NEW to ACTIVE, in the base table, will use a single write operation consuming one WCU; With the GSI, it will consume two WCUs, one to delete the NEW status and another to set it to ACTIVE, approaching to the <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/burst-adaptive-capacity.html#adaptive-capacity">1000 WCU limit per virtual partition</a>.</p> 
<p>Fortunately, we can use the status attribute as a sharding key, assuming that we will have five different statuses you can increase the throughput by up to 5x. Create a new GSI where the partition key is customer_id and status, and the sort key is order_date and amount.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td colspan="2"><strong>GSI PK</strong></td> 
   <td colspan="2"><strong>GSI SK</strong></td> 
   <td></td> 
   <td></td> 
   <td></td> 
  </tr> 
  <tr> 
   <td><strong>customer_id</strong></td> 
   <td><strong>status</strong></td> 
   <td><strong>order_date</strong></td> 
   <td><strong>amount</strong></td> 
   <td><strong>order_id</strong></td> 
   <td><strong>acc_type</strong></td> 
   <td><strong>org_id</strong></td> 
  </tr> 
  <tr> 
   <td>C#1A2B3C</td> 
   <td>ACTIVE</td> 
   <td>2025-11-04</td> 
   <td>200</td> 
   <td>KSUID1</td> 
   <td>A</td> 
   <td>OMEGA</td> 
  </tr> 
  <tr> 
   <td>C#1A2B3C</td> 
   <td>PENDING</td> 
   <td>2025-11-04</td> 
   <td>145</td> 
   <td>KSUID2</td> 
   <td>A</td> 
   <td>OMEGA</td> 
  </tr> 
  <tr> 
   <td>C#1A2B3C</td> 
   <td>PENDING</td> 
   <td>2025-11-04</td> 
   <td>110</td> 
   <td>KSUID3</td> 
   <td>B</td> 
   <td>BRAVO</td> 
  </tr> 
 </tbody> 
</table> 
<div class="hide-language"> 
 <pre><code class="lang-bash">GSI: OrdersByOrgAccountStatus
Partition Key 1: customer_id
Partition Key 2: status
Sort Key 2: order_date (equality condition)
Sort Key 3: amount (range condition)
</code></pre> 
</div> 
<h2>Best practices for using multi-attribute composite key</h2> 
<p>Design for your query patterns first. Before creating GSIs, identify your top 3-5 most popular query patterns, understand their frequency and performance requirements. Design your base table structure for uniqueness and use the GSI to optimize for composite key patterns, ensuring that your most common queries can be satisfied efficiently. With composite keys GSIs, you can adapt to new requirements by adding indexes with different key combinations, rather than restructuring your entire data model.</p> 
<p>Choose your key order carefully. The order of attributes in your GSI directly impacts which queries you can run. You don’t need to use four partition keys and four sort keys, choose the combination that matches your access patterns, whether that’s three partition keys with two sort keys, one partition key with three sort keys, or any other configuration.</p> 
<p>One of the most powerful aspects of composite keys GSIs is the support for native data types. Use Number types for timestamps, quantities, and numeric comparisons to enable proper sorting and mathematical operations. Use Boolean flags for binary states like active/inactive or enabled/disabled. Avoid converting values to strings unless necessary, as this eliminates the benefits of type-specific operations.</p> 
<p>Plan for scale from the start. Design <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-indexes-general-sparse-indexes.html" rel="noopener noreferrer" target="_blank">sparse indexes</a> whenever possible to reduce costs by only indexing items that have values for your specified key attributes. Choose your projection type strategically: use ALL for flexibility, KEYS_ONLY to reduce storage costs, or INCLUDE to project only the attributes you need. Implement <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html" rel="noopener noreferrer" target="_blank">time to live</a> (TTL) strategies in your base table to manage record size over time and prevent unbounded growth.</p> 
<h2>Conclusion and next steps</h2> 
<p>In this post you learned how to work with composite keys GSIs. With this new DynamoDB feature you can query different data types and attributes without needing to use workarounds like concatenating attributes. It is available now on all DynamoDB tables at no additional cost beyond standard GSI storage, throughput and features.Ready to simplify your DynamoDB data model? Follow these steps:</p> 
<ol> 
 <li><strong>Identify your query patterns</strong>: Review which queries currently use concatenated sort keys</li> 
 <li><strong>Design your GSI</strong>: Map your attributes to partition and sort key positions based on data hierarchies.</li> 
 <li><strong>Test thoroughly</strong>: Create a test table and validate your query patterns work as expected under anticipated load.</li> 
 <li><strong>Deploy to production</strong>: Add the new GSI to your production table and update your application code</li> 
 <li><strong>Monitor performance</strong>: Track GSI metrics in CloudWatch and optimize as needed</li> 
 <li><strong>Clean up</strong>: Remove legacy composite attributes that are used in the GSI.</li> 
</ol> 
<p>For detailed implementation guidance, see the <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html" rel="noopener noreferrer" target="_blank">How to use global secondary indexes in DynamoDB</a> and <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html" rel="noopener noreferrer" target="_blank">Best Practices for Data Modeling</a>.</p> 
<hr /> 
<h3>About the authors</h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Esteban Serna" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2025/11/04/image-8.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Esteban Serna</h3> 
  <p><a href="https://www.linkedin.com/in/tebanieo/" rel="noopener" target="_blank">Esteban</a> is a Principal DynamoDB Specialist Solutions Architect at AWS. He’s a database enthusiast with 16 years of experience. From deploying contact center infrastructure to falling in love with NoSQL, Esteban’s journey led him to specialize in distributed computing. Passionate about his work, Esteban loves nothing more than sharing his knowledge with others.</p>
 </div> 
</footer>
