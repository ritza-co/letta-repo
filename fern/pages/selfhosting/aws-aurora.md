# How Letta Builds Production-Ready AI Agents with Amazon Aurora PostgreSQL

AI agents require persistent memory to maintain context, learn from past interactions, and provide consistent responses over time. Without long-term memory, agents start each interaction from scratch, limiting their ability to build relationships with users or leverage historical context. This is particularly crucial in production environments where agents handle multiple concurrent users and need to maintain reliable state across sessions and system restarts.

Consider a customer service AI agent deployed at a large e-commerce company. Without long-term memory, the agent would need to ask customers to repeat their order numbers, shipping preferences, and past issues in every conversation. This creates a frustrating experience where loyal customers feel unrecognized and need to constantly provide context. With long-term memory, the agent can recall previous interactions, understand customer preferences, and maintain context across multiple support sessions – even when conversations span several days or weeks. This kind of persistent memory transforms customer service from transactional exchanges into meaningful, context-aware interactions.

[Letta](https://www.letta.com/), an open-source framework for building stateful AI agents, offers a powerful solution for creating agents with long-term memory. While Letta's default SQLite database, which stores session memories, works well for development, production deployments demand a more robust database solution. This is where [Amazon Aurora PostgreSQL-Compatible Edition](https://aws.amazon.com/rds/aurora/) shines.

In this post, we'll guide you through setting up [Aurora Serverless v2](https://aws.amazon.com/rds/aurora/serverless/) as a scalable, highly available PostgreSQL database repository for storing Letta long-term memory. You'll learn how to create an Aurora cluster in the cloud, configure Letta to connect to it, and deploy agents that persist their memory to Aurora. We'll also explore how to query the database directly to view agent state.

## Prerequisites

Before we begin, ensure you have:

1. An [AWS account](https://aws.amazon.com/free/) with permissions to create Aurora clusters and modify [security groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
2. [Docker](https://www.docker.com/get-started/) installed on your local machine
3. An [OpenAI API key](https://www.docker.com/get-started/) for Letta's language model integration
4. [PostgreSQL client tools](https://www.postgresql.org/download/) installed locally (psql version 12 or higher recommended)
5. Python 3.8+ with pip, or Node.js 16+ with npm

## Solution overview

Letta runs in a Docker container on your local machine and connects to Aurora PostgreSQL over the internet. Aurora stores all agent configuration, memory, and conversation history in PostgreSQL tables. The connection uses standard PostgreSQL wire protocol on port 5432.

### Amazon Aurora

The integration of Aurora PostgreSQL brings several powerful capabilities essential for production-ready AI agent memory systems. At its core, Aurora PostgreSQL provides native support for the pgvector extension, enabling efficient similarity searches across millions of vector embeddings, which are the numerical representation of past conversations. This capability is crucial for AI agents that need to quickly retrieve contextually relevant information. The system can handle vectors up to 16,384 dimensions, accommodating advanced AI models and their complex embedding requirements.

Performance is a key strength of this architecture. Aurora PostgreSQL delivers sub-10ms query latency for memory lookups and supports up to 15 read replicas to scale memory retrieval operations efficiently. This means your AI agents can access their memory rapidly, even under heavy loads. The system's storage capacity extends up to 128TB, providing ample space for extensive memory archives and long-term conversation history.

Reliability is ensured through Aurora's comprehensive durability features. The system maintains 6-way replication across three Availability Zones, dramatically reducing the risk of data loss. Your agents' memories are further protected by point-in-time recovery capabilities with up to 35 days of backup retention. The self-healing storage system continuously performs data integrity checks, ensuring the consistency of your agents' memory states.

Scalability is built into the foundation of Aurora Serverless v2. The system automatically scales based on workload, handling thousands of concurrent agent connections effortlessly. Storage scaling is equally dynamic, growing from 10GB to 128TB without any downtime, ensuring your agents never run out of memory space as they learn and interact.

### Letta

[Letta treats agents as persistent services](https://docs.letta.com/getting-started/core-concepts), maintaining state server-side and enabling agents to run independently, communicate with each other, and continue processing when clients are offline. For production workloads, [Letta supports horizontal scaling via Kubernetes](https://docs.letta.com/guides/selfhosting/performance), with configurable worker processes and database connection pooling. The [background execution mode](https://docs.letta.com/guides/agents/long-running) enables resumable streams that survive disconnects and allow load balancing by picking up streams started by other instances.

Production deployments support multi-tenancy with unlimited agents on [Pro and Enterprise plans](https://docs.letta.com/guides/cloud/plans), making Letta suitable for large-scale customer service and multi-user applications. The platform includes [enterprise features](https://docs.letta.com/guides/cloud/rbac) like SAML/OIDC SSO, role-based access control, and tool sandboxing, with [telemetry and performance monitoring](https://docs.letta.com/guides/server/otel) for tracking system metrics. While this guide demonstrates a local setup, production deployments typically use [cloud platforms](https://docs.letta.com/guides/server/remote) with HTTPS access and [security controls](https://docs.letta.com/guides/selfhosting).

Here's what you'll build:

1. An Aurora Serverless v2 cluster with the [pgvector](https://github.com/pgvector/pgvector) extension for embedding storage
2. A security group configuration that allows your IP address to connect on port 5432
3. A Letta Docker container configured with `LETTA_PG_URI` pointing to Aurora
4. Working AI agents that persist all state to Aurora instead

This guide demonstrates the integration using Aurora Serverless v2 with minimal capacity settings suitable for development and testing environments.

## Setting up Amazon Aurora Serverless v2

You'll create an Aurora PostgreSQL cluster configured for external access from your local machine. Aurora Serverless v2 automatically scales capacity based on workload, making it cost-effective for development and testing.

### Create the Aurora cluster

To create the Aurora cluster:

1. Open the [Amazon RDS console](https://console.aws.amazon.com/rds/).
2. In the navigation pane, choose **Databases**.
3. Click **Create database**.

   ![Creating a database in RDS console](../../images/aws-aurora-integration/rds-create-database.png)

4. For **Engine options**, select **Aurora (PostgreSQL Compatible)**.
5. For **Engine version**, select **Aurora PostgreSQL (Compatible with PostgreSQL 17.4)** or your preferred version.
6. Under **Templates**, select **Dev/Test** to optimize for lower costs.
7. Under **Settings**, configure:
   - **DB cluster identifier**: Enter a name such as `letta-aurora-cluster`
   - **Master username**: Keep the default `postgres`
   - **Master password**: Enter a simple alphanumeric password
8. Under **Instance configuration**, select **Serverless v2**.
9. For **Capacity range**, set:
   - **Minimum capacity (ACUs)**: 0.5
   - **Maximum capacity (ACUs)**: 1

   ACUs (Aurora Capacity Units) are the measurement unit for database compute capacity in Aurora Serverless v2. Each ACU is a combination of approximately 2 GB of memory, corresponding CPU and networking capabilities. For example, 0.5 ACUs provides 1 GB of memory, while 32 ACUs provides 64 GB of memory with proportional compute resources. Your Aurora [database charges](https://aws.amazon.com/rds/aurora/pricing/) are based on the ACU usage per second. For development and testing, starting with 0.5-1 ACU is sufficient. Production workloads typically require higher ACU ranges based on your application's memory and processing needs. The beauty of Aurora Serverless v2 is that it automatically adjusts capacity within your specified ACU range based on actual database load. Aurora can even can scale to a minimum capacity of zero ACUs, which eliminates all compute charges in time periods where your cluster is not being used.

10. Under **Connectivity**, configure:
    - **Public access**: Choose **Yes**
    - **VPC security group**: Select **Create new**
    - **New VPC security group name**: Enter `letta-aurora-sg`

11. Choose **Create database**.

The cluster creation takes approximately 10-15 minutes. While Aurora creates the cluster, you can proceed to configure the security group.

### Configure security group access

Aurora clusters require security group configuration to allow external connections. By default, the security group blocks all incoming traffic.

To configure the security group:

1. Open the [EC2 console](https://console.aws.amazon.com/ec2/).
2. In the navigation pane, under **Network & Security**, choose **Security Groups**.
3. Open the **Inbound rules** tab. Find the security group attached to your Aurora cluster (`letta-aurora-sg`) and select it.
4. Click **Edit inbound rules**.

   ![Navigating to edit inbound rules](../../images/aws-aurora-integration/edit-inbound-rules.png)

5. Choose **Add rule** and configure:
   - **Type**: PostgreSQL
   - **Protocol**: TCP (automatically set)
   - **Port range**: 5432 (automatically set)
   - **Source**: Choose one of the following options:
     - **My IP**: Automatically detects and allows your current IP address (recommended for testing)
     - **Custom**: Enter a specific IP address with /32 suffix for single IP access

   ![Configuring inbound rule with My IP](../../images/aws-aurora-integration/configure-my-ip-rule.png)

6. Click **Save rules**.

Your security configuration now allows PostgreSQL connections from your IP address to the Aurora cluster.

### Get the cluster endpoint

After the cluster status shows **Available**, retrieve the connection endpoint:

1. In the [Amazon RDS console](https://console.aws.amazon.com/rds/), select your cluster.

   ![Selecting the created database in RDS console](../../images/aws-aurora-integration/rds-select-database.png)

2. In the **Connectivity & security** tab, find the **Endpoints** section.
3. Copy the **Writer instance endpoint**. It looks similar to:
   - letta-aurora-cluster.cluster-abc123def456.us-east-1.rds.amazonaws.com

   ![Copying the writer endpoint from RDS console](../../images/aws-aurora-integration/rds-copy-endpoint.png)

You'll use this endpoint to construct the PostgreSQL connection string.

### Install the pgvector extension

Letta uses the pgvector extension to store vector embeddings for agent memory. You need to manually enable this extension before connecting Letta.

To install pgvector:

1. Connect to Aurora using the psql command-line tool:

   ```bash
   psql -h letta-aurora-cluster.cluster-abc123def456.us-east-1.rds.amazonaws.com \
        -U postgres \
        -d postgres
   ```

2. When prompted, enter the master password you configured during cluster creation.
3. After successful connection, you'll see the PostgreSQL prompt:

   ```
   postgres=>
   ```

4. Create the pgvector extension:

   ```sql
   CREATE EXTENSION vector;
   ```

5. Verify the installation:

   ```sql
   SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';
   ```

   You should see:

   ```
    extname | extversion
   ---------+------------
    vector  | 0.8.0
   ```

6. Exit psql:

   ```sql
   \q
   ```

Your Aurora cluster is now ready for Letta connections.

## Connecting Letta to Aurora

With Aurora configured, you can now run Letta and connect it to your cluster using the `LETTA_PG_URI` environment variable. Letta's Docker image automatically detects the external PostgreSQL connection and runs database migrations on startup.

### Construct the connection string

The `LETTA_PG_URI` follows the standard PostgreSQL connection string format:

```
postgresql://USERNAME:PASSWORD@ENDPOINT:PORT/DATABASE
```

Using the values from your Aurora cluster:

```
postgresql://postgres:LettaTest2024@letta-aurora-cluster.cluster-abc123def456.us-east-1.rds.amazonaws.com:5432/postgres
```

Replace the endpoint with your actual cluster endpoint and adjust the password if you used a different one.

### Run Letta with Docker

To start Letta connected to Aurora, run the Docker container with your Aurora connection string and OpenAI API key:

```bash
docker run -p 8283:8283 \
  -e LETTA_PG_URI='postgresql://postgres:YOUR_PASSWORD@YOUR_CLUSTER_ENDPOINT:5432/postgres' \
  -e OPENAI_API_KEY='your-openai-api-key' \
  letta/letta:latest
```

Replace `YOUR_PASSWORD` with your Aurora master password, `YOUR_CLUSTER_ENDPOINT` with your cluster endpoint, and `your-openai-api-key` with your OpenAI API key.

Watch for the migration output. You should see:

```
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 9a505cc7eca9, Create a baseline migrations
```

This confirms Letta detected the PostgreSQL connection and created the necessary database schema.

When you see the following output, the server is ready:

```
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8283
```

Letta is now running and connected to Aurora. All agent data will persist to your Aurora cluster instead of local storage.

### Create and test an agent

To verify the connection works, create an agent and send it a message. Choose either Python or TypeScript based on your preferred language.

#### Using Python

1. In a new terminal, install the Letta Python client:

   ```bash
   pip install letta-client
   ```

2. Create a Python script called `test_aurora_agent.py`:

   ```python
   from letta_client import Letta

   # Connect to Letta server with extended timeout
   client = Letta(base_url="http://localhost:8283", timeout=600)

   # Create a new agent
   agent = client.agents.create(
       name="aurora_test_agent",
       memory_blocks=[
           {
               "label": "human",
               "value": "Name: Test User"
           },
           {
               "label": "persona",
               "value": "I am a helpful AI assistant with persistent memory stored in Aurora."
           }
       ],
       model="openai/gpt-4o-mini",
       embedding="openai/text-embedding-3-small"
   )

   print(f"Created agent: {agent.id}")

   # Send a message to the agent
   response = client.agents.messages.create(
       agent_id=agent.id,
       messages=[{
           "role": "user",
           "content": "Hello! Can you remember that my favorite color is blue?"
       }]
   )

   # Extract and display the assistant's response
   for message in response.messages:
       if message.message_type == "assistant_message":
           content = message.content
           if isinstance(content, str):
               print(f"\nAgent: {content}")
           elif isinstance(content, list):
               for item in content:
                   if item.get("type") == "text":
                       print(f"\nAgent: {item.get('text', '')}")
   ```

3. Run the script:

   ```bash
   python test_aurora_agent.py
   ```

4. You should see output similar to:

   ```
   Created agent: agent-abc123

   Agent: Of course! I've made a note that your favorite color is blue. I'll remember that for our future conversations.
   ```

#### Using TypeScript

1. Initialize a Node.js project and install dependencies:

   ```bash
   npm init -y
   npm install @letta-ai/letta-client tsx
   npm install --save-dev @types/node
   ```

2. Update your `package.json` to use ES modules by adding:

   ```json
   {
     "type": "module"
   }
   ```

3. Create a TypeScript script called `test_aurora_agent.ts`:

   ```typescript
   import { LettaClient } from '@letta-ai/letta-client';

   async function main() {
     // Connect to Letta server
     const client = new LettaClient({
       baseUrl: 'http://localhost:8283',
     });

     // Create an agent
     const agent = await client.agents.create({
       name: 'aurora_test_agent',
       model: 'openai/gpt-4o-mini',
       embedding: 'openai/text-embedding-3-small',
       memoryBlocks: [
         {
           label: 'persona',
           value: 'I am a helpful AI assistant with persistent memory stored in Aurora.',
         },
         {
           label: 'human',
           value: 'Name: Test User',
         },
       ],
     });

     console.log(`Created agent: ${agent.id}`);

     // Send a message with extended timeout
     const response = await client.agents.messages.create(
       agent.id,
       {
         messages: [
           { role: 'user', content: 'Hello! Can you remember that my favorite color is blue?' }
         ],
       },
       {
         timeoutInSeconds: 600, // 10 minutes
       }
     );

     // Extract and print assistant's response
     for (const message of response.messages) {
       if (message.messageType === 'assistant_message') {
         console.log(`\nAgent: ${(message as any).content}`);
       }
     }
   }

   main().catch(console.error);
   ```

   > **Note**: You may see validation warnings about `tool_return_message` when running this script. These are internal SDK type validation messages. The warnings don't affect functionality. To suppress these warnings, redirect stderr: `npx tsx test_aurora_agent.ts 2>/dev/null`

4. Run the script:

   ```bash
   npx tsx test_aurora_agent.ts
   ```

   Or to suppress validation warnings:

   ```bash
   npx tsx test_aurora_agent.ts 2>/dev/null
   ```

5. You should see output similar to:

   ```
   Created agent: agent-abc123

   Agent: Of course! I've made a note that your favorite color is blue. I'll remember that for our future conversations.
   ```

The agent response confirms that Letta successfully created an agent and processed your message. All of this data is now stored in Aurora.

## Viewing agent state in Aurora

Now that you have agents running and storing data in Aurora, you can connect directly to the database to inspect how Letta organizes agent state, memory, and conversations.

### Connect to Aurora

Using the same psql connection from earlier:

```bash
psql -h letta-aurora-cluster.cluster-abc123def456.us-east-1.rds.amazonaws.com \
     -U postgres \
     -d postgres
```

### Explore the database schema

Letta creates 42 tables to manage agents, memory, messages, and associated metadata. To view all tables:

```sql
\dt
```

You should see output showing tables including `agents`, `messages`, `organizations`, `users`, `block`, `sources`, and others:

```
                     List of relations
 Schema |             Name              | Type  |  Owner
--------+-------------------------------+-------+----------
 public | agents                        | table | postgres
 public | block                         | table | postgres
 public | messages                      | table | postgres
 public | organizations                 | table | postgres
 public | users                         | table | postgres
 ...
(42 rows)
```

The key tables for understanding agent state are:

- **agents**: Stores agent configuration and metadata
- **messages**: Contains all conversation messages with full content as JSON
- **organizations** and **users**: Manages multi-tenant access control
- **block** and **block_history**: Stores memory blocks and their revision history
- **sources** and **source_passages**: Contains data sources and their embeddings for retrieval

### View agent information

To see all agents in your database:

```sql
SELECT id, name, created_at FROM agents;
```

Output example:

```
                     id                     |       name        |          created_at
--------------------------------------------+-------------------+-------------------------------
 agent-29cdd087-44a7-4dd4-b34e-d38dd5ef2935 | aurora_test_agent | 2025-10-27 06:30:56.078296+00
(1 row)
```

### View agent messages

To see the conversation history for a specific agent, use the `content` column which stores messages as JSON (the `text` column is empty in current Letta versions):

```sql
SELECT id, role, content::text, created_at
FROM messages
WHERE agent_id = 'agent-29cdd087-44a7-4dd4-b34e-d38dd5ef2935'
ORDER BY created_at
LIMIT 5;
```

Output example:

```
                      id                      |   role    |                              content                              |          created_at
----------------------------------------------+-----------+-------------------------------------------------------------------+-------------------------------
 message-9beba3d8-61e3-4eb9-a0b9-94d5745b9431 | system    | [{"type": "text", "text": "<base_instructions>\nYou are Letta...  | 2025-10-27 06:31:02.505196+00
 message-64be4f18-ccaa-461c-af21-502372f9a081 | assistant | [{"type": "text", "text": "Bootup sequence complete. Persona...   | 2025-10-27 06:31:02.515623+00
 message-f68f492e-c126-477d-a78a-f015036e07a0 | tool      | [{"type": "text", "text": "{\n  \"status\": \"OK\",\n  \"message\... | 2025-10-27 06:31:02.516277+00
 message-b9720ed1-6a1f-4827-ad11-60a7dd62e032 | user      | [{"type": "text", "text": "{\n  \"type\": \"login\",\n  \"last_... | 2025-10-27 06:31:02.516304+00
 message-4cc57dc7-1952-4e24-a202-f93dfb23004c | user      | [{"type": "text", "text": "Hello! Can you remember that my...     | 2025-10-27 06:31:45.689208+00
(5 rows)
```

The messages table stores the complete conversation flow as JSON, allowing you to trace exactly how agents process and respond to user input.

### Examine message content structure

Letta stores message content as JSON. To view the detailed structure of an assistant message:

```sql
SELECT content::text
FROM messages
WHERE agent_id = 'agent-29cdd087-44a7-4dd4-b34e-d38dd5ef2935'
  AND role = 'assistant'
LIMIT 1;
```

Output example:

```
                                                            content
--------------------------------------------------------------------------------------------------------------------------------
 [{"type": "text", "text": "Bootup sequence complete. Persona activated. Testing messaging functionality.", "signature": null}]
(1 row)
```

This shows the JSON structure of assistant messages. Each message is stored as an array of content objects with `type`, `text`, and optional `signature` fields. System messages contain the full agent instructions including memory blocks, while assistant and user messages contain the conversation content.

### Understanding the schema for embeddings and data sources

Letta uses the pgvector extension to store embedding vectors for semantic memory search and retrieval-augmented generation (RAG). The database includes tables specifically designed for vector storage:

```sql
-- Check archival passages (populated when agents use archival memory)
SELECT COUNT(*) FROM archival_passages;

-- Check source passages (populated when you attach data sources to agents)
SELECT COUNT(*) FROM source_passages;
```

For a newly created agent with basic conversation history, both tables will show 0 records. These tables are populated when you:

- Attach documents or files to an agent as data sources
- Use archival memory features for long-term storage
- Implement retrieval-augmented generation workflows

The `source_passages` table stores embeddings for external data sources, while `archival_passages` stores embeddings for the agent's archival memory system. Both utilize pgvector's vector data type for efficient similarity search.

## Clean up

When you've finished exploring the integration, you can delete the Aurora cluster and associated resources to avoid ongoing charges.

### Delete the Aurora cluster

To delete your Aurora cluster:

1. Open the [Amazon RDS console](https://console.aws.amazon.com/rds/).
2. In the navigation pane, choose **Databases**.
3. Select your cluster (such as `letta-aurora-cluster`).
4. Choose **Actions**, then choose **Delete**.
5. For **Create final snapshot**, choose **No** (this is a test environment).
6. Enter `delete me` in the confirmation field.
7. Choose **Delete**.

The deletion process takes several minutes.

### Delete the security group

After the cluster is deleted:

1. Open the [EC2 console](https://console.aws.amazon.com/ec2/).
2. In the navigation pane, choose **Security Groups**.
3. Select the security group you created (such as `letta-aurora-sg`).
4. Choose **Actions**, then choose **Delete security groups**.
5. Choose **Delete**.

## Conclusion

In this post, you configured Letta to use Amazon Aurora PostgreSQL-Compatible Edition as a managed database backend. You created an Aurora Serverless v2 cluster, configured security group access, installed the pgvector extension, and connected Letta using the `LETTA_PG_URI` environment variable. You then created AI agents that persist their state to Aurora and queried the database to view agent conversations and memory.

This integration enables production deployments of Letta with the scalability, durability, and high availability that Aurora provides. For production use, consider implementing additional security measures such as IAM database authentication, encryption at rest, and restricting security group access to specific IP ranges or VPC configurations.

To learn more about Letta, visit the [Letta documentation](https://docs.letta.com/). For more information about Aurora PostgreSQL, see the [Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html).
