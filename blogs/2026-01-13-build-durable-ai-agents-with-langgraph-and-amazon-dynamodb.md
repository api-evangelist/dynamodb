---
title: "Build durable AI agents with LangGraph and Amazon DynamoDB"
url: "https://aws.amazon.com/blogs/database/build-durable-ai-agents-with-langgraph-and-amazon-dynamodb/"
date: "Tue, 13 Jan 2026 18:00:11 +0000"
author: "Lee Hannigan"
feed_url: "https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/feed/"
---
<p>I’ve been fascinated by the rapid evolution of AI agents. Over the past year, I’ve watched them grow from simple chatbots into sophisticated systems that can reason through complex problems, make decisions, and maintain context across long conversations. Yet an agent is only as good as its memory.</p> 
<p>In this post we show you how to build production-ready AI agents with durable state management using <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> and <a href="https://langchain-ai.github.io/langgraph/" rel="noopener noreferrer" target="_blank">LangGraph</a> with the new <a href="https://github.com/langchain-ai/langchain-aws/blob/main/libs/langgraph-checkpoint-aws/docs/dynamodb/DynamoDBSaver.md" rel="noopener noreferrer" target="_blank">DynamoDBSaver</a> connector, a LangGraph checkpoint library maintained by AWS for Amazon DynamoDB. It provides a production-ready persistence layer built specifically for DynamoDB and LangGraph that stores agent state with intelligent handling of payloads based on their size.</p> 
<p>You’ll learn how this implementation can give your agents the persistence they need to scale, recover from failures, and maintain long-running workflows.</p> 
<h2><strong>A quick look at Amazon DynamoDB</strong></h2> 
<p>Amazon DynamoDB is a serverless, fully managed, distributed NoSQL database with single-digit millisecond performance at any scale. You can store structured or semi-structured data, query it with consistent millisecond latency, and scale automatically without managing servers or infrastructure.Because DynamoDB is built for low latency and high availability, it is often used to store session data, user profiles, metadata, or application state. These same qualities make it an ideal choice for storing checkpoints and thread metadata for AI agents.</p> 
<h2><strong>Introducing LangGraph</strong></h2> 
<p>LangGraph is an open source framework from <a href="https://www.langchain.com/" rel="noopener noreferrer" target="_blank">LangChain</a> designed for building complex, graph-based AI workflows. Instead of chaining prompts and functions in a straight line, LangGraph lets you define nodes that can branch, merge, and loop. Each node performs a task, and edges control the flow between them.</p> 
<p>LangGraph introduces several key concepts:</p> 
<ul> 
 <li><strong>Threads</strong>: A thread is a unique identifier assigned to each checkpoint that contains the accumulated state of a sequence of runs. When a graph executes, its state persists to the thread, which requires specifying a thread_id in the config (<code>{"configurable": {"thread_id": "1"}}</code>). Threads must be created before execution to persist state.</li> 
 <li><strong>Checkpoints</strong>: A checkpoint is a snapshot of the graph state saved at each super-step, represented by a StateSnapshot object containing config, metadata, state channel values, next nodes to execute, and task information (including errors and interrupt data). Checkpoints are persisted and can restore thread state later. For example, a simple two-node graph creates four checkpoints: an empty checkpoint at <code>START</code>, one with user input before node_a, one with node_a’s output before node_b, and a final one with node_b’s output at <code>END</code>.</li> 
 <li><strong>Persistence</strong>: Persistence determines where and how checkpoints are stored (such as, in-memory, database, or external storage) using a checkpointer implementation. The checkpointer saves thread state at each super-step and enables retrieval of historical states, allowing graphs to resume from checkpoints or restore previous execution states.</li> 
</ul> 
<p>Persistence is what enables advanced features such as human-in-the-loop review, replay, resumption after failure, and time travel between states.</p> 
<p><a href="https://docs.langchain.com/oss/python/langgraph/persistence" rel="noopener noreferrer" target="_blank">InMemorySaver</a> is LangGraph’s built-in checkpointing mechanism that stores conversation state and graph execution history in memory, enabling features like persistence, time-travel debugging, and human-in-the-loop workflows. You can use <code>InMemorySaver</code> for fast prototyping, state exists only in memory and is lost when your application restarts.</p> 
<p>The following image shows LangGraph’s checkpointing architecture, where a high-level workflow (super-step) executes through nodes from <code>START</code> to <code>END</code> while a checkpointer continuously saves state snapshots to memory (<code>InMemorySaver</code>):</p> 
<p><img alt="In memory process" class="aligncenter size-full wp-image-68115" height="783" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/08/lhnng-image-1.png" width="1729" /></p> 
<h2><strong>Why persistence matters</strong></h2> 
<p>By default, LangGraph stores checkpoints in memory using the <code>InMemorySaver</code>. This is great for experimentation because it requires no setup and offers instant read and write access.</p> 
<p>However, in memory storage has two major limitations. It is ephemeral and local. When the process stops, the data is lost. If you run multiple workers, each instance keeps its own memory. You cannot resume a session that started elsewhere, and you cannot recover if a workflow crashes halfway.</p> 
<p>For production environments, this is not acceptable. You need a persistent, fault-tolerant store that allows agents to resume where they left off, scale across nodes, and retain history for analysis or audit. That is where the <code>DynamoDBSaver</code> comes in.</p> 
<p>Imagine a scenario where you’re building a customer support agent that handles complex, multi-step inquiries. A customer asks about their order, the agent retrieves information, generates a response, and waits for human approval before sending a response.</p> 
<p>But what happens when:</p> 
<ul> 
 <li>Your server times out mid-workflow?</li> 
 <li>You need to scale to multiple workers?</li> 
 <li>The customer comes back hours later to continue the conversation?</li> 
 <li>You want to audit the agent’s decision-making process?</li> 
</ul> 
<p>With in-memory storage, you’re out of luck. The moment your process stops, everything vanishes. Each worker maintains its own isolated state. There’s no way to resume, replay, or review what happened.</p> 
<h2><strong>Introducing DynamoDBSaver</strong></h2> 
<p>The <a href="https://github.com/langchain-ai/langchain-aws/tree/main/libs/langgraph-checkpoint-aws" rel="noopener noreferrer" target="_blank">langgraph-checkpoint-aws</a> library provides a persistence layer built specifically for AWS. <code>DynamoDBSaver</code> stores lightweight checkpoint metadata in DynamoDB and uses Amazon S3 for large payloads.</p> 
<p>Here is how it works:</p> 
<ol> 
 <li><strong>Small checkpoints</strong> (&lt; 350 KB): Stored directly in DynamoDB as serialized items with metadata like <code>thread_id</code>, <code>checkpoint_id</code>, timestamps, and state</li> 
 <li><strong>Large checkpoints</strong> (≥ 350 KB): State is uploaded to S3, and DynamoDB stores a reference pointer to the S3 object</li> 
 <li><strong>Retrieval</strong>: When resuming, the saver fetches metadata from DynamoDB and transparently loads large payloads from S3</li> 
</ol> 
<p>This design provides durability, scalability, and efficient handling of both small and large states without hitting the DynamoDB item size limit.</p> 
<p><code>DynamoDBSaver</code> includes built-in features to help you manage costs and data lifecycle:</p> 
<ul> 
 <li>Time-to-Live (<code>ttl_seconds</code>) enables automatic expiration of checkpoints at specified intervals. Old thread states are cleaned up without manual intervention, ideal for temporary workflows, testing environments, or applications where a historical state beyond a certain age has no value.</li> 
 <li>Compression (<code>enable_checkpoint_compression</code>) reduces checkpoint size before storage by serializing and compressing state data, which lowers both DynamoDB write costs and S3 storage costs while maintaining full state fidelity upon retrieval.</li> 
</ul> 
<p>Together, these features help provide fine-grained control over your persistence layer’s operational costs and storage footprint, allowing you to balance durability requirements with budget constraints as your application scales.</p> 
<h2><strong>Getting started</strong></h2> 
<p>Let’s build a practical example showing how to persist agent state across executions and retrieve historical checkpoints.</p> 
<h3><strong>Prerequisites</strong></h3> 
<p>Before we begin, you’ll need to set up the required AWS resources:</p> 
<ul> 
 <li><strong>DynamoDB table</strong>: The <code>DynamoDBSaver</code> requires a table to store checkpoint metadata. The table must have a partition key named PK (String) and a sort key named SK (String).</li> 
 <li><strong>S3 bucket (optional)</strong>: If your checkpoints may exceed 350 KB, provide an S3 bucket for large payload storage. The saver will automatically route oversized states to S3 and store references in DynamoDB.</li> 
</ul> 
<p>You can use the <a href="https://docs.aws.amazon.com/cdk/v2/guide/home.html" rel="noopener noreferrer" target="_blank">AWS Cloud Development Kit </a>(AWS CDK) to define these resources:</p> 
<div class="hide-language"> 
 <pre><code class="lang-ts">const table = new dynamodb.Table(this, 'CheckpointTable', {
    tableName: 'my_langgraph_checkpoints_table',
    partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
    sortKey: { name: 'SK', type: dynamodb.AttributeType.STRING },
    timeToLiveAttribute: 'ttl',
    removalPolicy: cdk.RemovalPolicy.DESTROY,
});

const bucket = new s3.Bucket(this, 'CheckpointBucket', {
    bucketName: 'amzn-s3-demo-bucket',
    encryption: s3.BucketEncryption.S3_MANAGED,
    blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
    removalPolicy: cdk.RemovalPolicy.DESTROY
});</code></pre> 
</div> 
<p>Your application needs the following <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management </a>(AWS IAM) permissions to use <code>DynamoDBSaver</code> as LangGraph checkpoint storage:</p> 
<h4><strong>DynamoDB Table Access:</strong></h4> 
<ul> 
 <li><code>dynamodb:GetItem</code> – Retrieve individual checkpoints</li> 
 <li><code>dynamodb:PutItem</code> – Store new checkpoints</li> 
 <li><code>dynamodb:Query</code> – Search for checkpoints by thread ID</li> 
 <li><code>dynamodb:BatchGetItem</code> – Retrieve multiple checkpoints efficiently</li> 
 <li><code>dynamodb:BatchWriteItem</code> – Store multiple checkpoints in a single operation</li> 
</ul> 
<h4><strong>S3 Object Operations (for checkpoints larger than 350KB):</strong></h4> 
<ul> 
 <li><code>s3:PutObject</code> – Upload checkpoint data</li> 
 <li><code>s3:GetObject</code> – Retrieve checkpoint data</li> 
 <li><code>s3:DeleteObject</code> – Remove expired checkpoints</li> 
 <li><code>s3:PutObjectTagging</code> – Tag objects for lifecycle management</li> 
</ul> 
<h4><strong>S3 Bucket Configuration:</strong></h4> 
<ul> 
 <li><code>s3:GetBucketLifecycleConfiguration</code> – Read lifecycle rules</li> 
 <li><code>s3:PutBucketLifecycleConfiguration</code> – Configure automatic data expiration</li> 
</ul> 
<h3><strong>Installation</strong></h3> 
<p>Install LangGraph and the AWS checkpoint storage library using pip:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">pip install langgraph langgraph-checkpoint-aws</code></pre> 
</div> 
<h3><strong>Basic setup</strong></h3> 
<p>Configure the DynamoDB checkpoint saver with your table and optional S3 bucket for large checkpoints:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">from langgraph.graph import StateGraph, END
from langgraph_checkpoint_aws import DynamoDBSaver 
from typing import TypedDict, Annotatedimport operator

# Define your state
class State(TypedDict):
    foo: str
    bar: Annotated[list[str], add]

# Configure DynamoDB persistence
checkpointer = DynamoDBSaver(
    table_name="my_langgraph_checkpoints_table",
    region_name="us-east-1",
    ttl_seconds=86400 * 30,  # 30 days
    enable_checkpoint_compression=True,
    s3_offload_config={
        "bucket_name": "amzn-s3-demo-bucket", 
    }
)</code></pre> 
</div> 
<h3><strong>Building the workflow</strong></h3> 
<p>Create your graph and compile it with the checkpointer to enable persistent state across invocations:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python"># thread_id for session
THREAD_ID = "99"

workflow = StateGraph(State)
workflow.add_node(node_a)
workflow.add_node(node_b)
workflow.add_edge(START, "node_a")
workflow.add_edge("node_a", "node_b")
workflow.add_edge("node_b", END)

graph = workflow.compile(checkpointer=checkpointer)

config: RunnableConfig = {"configurable": {"thread_id": THREAD_ID}}

graph.invoke({"foo": "", "bar": []}, config)</code></pre> 
</div> 
<h3><strong>Obtaining state</strong></h3> 
<p>Retrieve the current state or access previous checkpoints for time-travel debugging:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python"># get the latest state snapshot
config = {"configurable": {"thread_id": THREAD_ID}}
latest_checkpoint = graph.get_state(config)
print(latest_checkpoint)

# get a state snapshot for a specific checkpoint_id
checkpoint_id = latest_checkpoint.config.get("configurable", {}).get("checkpoint_id")
config = {"configurable": {"thread_id": THREAD_ID, "checkpoint_id": checkpoint_id}}
specific_checkpoint = graph.get_state(config)
print(specific_checkpoint)</code></pre> 
</div> 
<h3><strong>Real-world use cases</strong></h3> 
<h4><strong>1. Human-in-the-loop review</strong></h4> 
<p>For sensitive operations (financial transactions, legal documents, medical advice), you can pause workflows for human oversight:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python"># Agent generates a response
workflow.invoke({"query": "Approve my loan"}, config)

# Human reviews in a separate process/UI
# Checkpoint is safely stored in DynamoDB
# After approval, resume
workflow.invoke({"approved": True}, config)</code></pre> 
</div> 
<h4><strong>2. Failure recovery</strong></h4> 
<p>In production systems, failures happen. Network interruptions, API timeouts, or transient errors can stop execution mid-way.</p> 
<p>With in-memory checkpoints, you lose progress. With <code>DynamoDBSaver</code>, the workflow can query the last successful checkpoint and resume from there. This helps reduce re-computation, speed up recovery, and improve reliability.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">try:
    workflow.invoke({"input": "complex query"}, config)
except Exception as e:
    # Log error, alert ops team
    pass

# Later, retry from the last successful checkpoint
# No need to re-execute completed steps
workflow.invoke({}, config)</code></pre> 
</div> 
<h4><strong>3. Long-running conversations</strong></h4> 
<p>Some workflows span hours or days. The durability of DynamoDB makes sure conversations persist:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python"># Day 1: Customer starts inquiry
workflow.invoke({"messages": ["I need help"]}, config)
# Day 2: Customer provides more info
workflow.invoke({"messages": ["Here's my account number"]}, config)
# Day 3: Agent completes the task
workflow.invoke({"action": "resolve"}, config)</code></pre> 
</div> 
<p>Moving from prototype to production is as simple as changing your checkpointer. Replace <code>MemorySaver</code> with <code>DynamoDBSaver</code> to gain persistent, scalable state management:</p> 
<p><img alt="DynamoDB process" class="aligncenter size-full wp-image-68116" height="980" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/08/lhnng-image-2.png" width="1729" /></p> 
<h2><strong>Clean up</strong></h2> 
<p>To avoid incurring ongoing charges, delete the resources you created:</p> 
<p>If you used AWS CDK to deploy, run the following command:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">cdk destroy</code></pre> 
</div> 
<p>If you used the CLI, run the following commands:</p> 
<ul> 
 <li>Delete the DynamoDB table: 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws dynamodb delete-table --table-name my_langgraph_checkpoints_table</code></pre> 
  </div> </li> 
 <li>Empty and delete the Amazon S3 bucket: 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws s3 rm s3://amzn-s3-demo-bucket --recursive
aws s3 rb s3://amzn-s3-demo-bucket</code></pre> 
  </div> </li> 
</ul> 
<h2><strong>Conclusion</strong></h2> 
<p>LangGraph makes it straightforward to build intelligent, stateful agents. <code>DynamoDBSaver</code> makes it safe to run them in production.</p> 
<p>By integrating <code>DynamoDBSaver</code> into your LangGraph applications, you can gain durability, scalability, and the ability to resume complex workflows from a specific point in time. You can build systems that involve human oversight, maintain long-running sessions, and recover gracefully from interruptions.</p> 
<h2><strong>Get Started Today</strong></h2> 
<p>Start with in-memory checkpoints while prototyping. When you’re ready to go live, switch to <code>DynamoDBSaver</code> and let your agents remember, recover, and scale with confidence. Install the library with pi<code>p install langgraph-checkpoint-aws</code>.</p> 
<p>Learn more about the <code>DynamoDBSaver</code> on the <a href="https://github.com/langchain-ai/langchain-aws/blob/main/libs/langgraph-checkpoint-aws/docs/dynamodb/DynamoDBSaver.md" rel="noopener noreferrer" target="_blank">langgraph-checkpoint-aws documentation</a> to see the available configuration options.</p> 
<p>For production workloads, consider hosting your LangGraph agents using <a href="https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agents-tools-runtime.html" rel="noopener noreferrer" target="_blank">Amazon Bedrock AgentCore Runtime</a>. AgentCore provides a fully managed runtime environment that handles scaling, monitoring, and infrastructure management, allowing you to focus on building agent logic while AWS manages the operational complexity.</p> 
<hr /> 
<h3>About the authors</h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Lee Hannigan" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/01/08/lhnng-image-3.png" width="120" />
  </div> 
  <h3 class="lb-h4">Lee Hannigan</h3> 
  <p><a href="https://www.linkedin.com/in/lee-hannigan/" rel="noopener" target="_blank">Lee</a> is a Sr. DynamoDB Database Engineer based in Donegal, Ireland. He brings a wealth of expertise in distributed systems, with a strong foundation in big data and analytics technologies. In his role, Lee focuses on advancing the performance, scalability, and reliability of DynamoDB while helping customers and internal teams make the most of its capabilities.</p>
 </div> 
</footer>
