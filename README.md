# Programmatic-tool-calling
Programmatic tool calling allows Claude to write code that calls your tools programmatically within a code execution container, rather than requiring round trips through the model for each tool invocation.
This reduces latency for multi-tool workflows and decreases token consumption by allowing Claude to filter or process data before it reaches the model's context window. On agentic search benchmarks like BrowseComp and DeepSearchQA, which test multi-step web research and complex information retrieval, adding programmatic tool calling on top of basic search tools was the key factor that fully unlocked agent performance.

The difference compounds fast in real workflows. Consider checking budget compliance across 20 employees: the traditional approach requires 20 separate model round-trips, pulling thousands of expense line items into the context along the way. With programmatic tool calling, a single script runs all 20 lookups, filters the results, and returns only the employees who exceeded their limits, shrinking what Claude needs to reason over from hundreds of kilobytes down to a handful of lines.

Quick start
Here's a simple example where Claude programmatically queries a database multiple times and aggregates results:


    --header "x-api-key: $ANTHROPIC_API_KEY" \
    --header "anthropic-version: 2023-06-01" \
    --header "content-type: application/json" \
    --data '{
        "model": "claude-opus-4-6",
        "max_tokens": 4096,
        "messages": [
            {
                "role": "user",
                "content": "Query sales data for the West, East, and Central regions, then tell me which region had the highest revenue"
            }
        ],
        "tools": [
            {
                "type": "code_execution_20260120",
                "name": "code_execution"
            },
            {
                "name": "query_database",
                "description": "Execute a SQL query against the sales database. Returns a list of rows as JSON objects.",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "sql": {
                            "type": "string",
                            "description": "SQL query to execute"
                        }
                    },
                    "required": ["sql"]
                },
                "allowed_callers": ["code_execution_20260120"]
            }
        ]
    }'
How programmatic tool calling works
When you configure a tool to be callable from code execution and Claude decides to use that tool:

Claude writes Python code that invokes the tool as a function, potentially including multiple tool calls and pre/post-processing logic
Claude runs this code in a sandboxed container via code execution
When a tool function is called, code execution pauses and the API returns a tool_use block
You provide the tool result, and code execution continues (intermediate results are not loaded into Claude's context window)
Once all code execution completes, Claude receives the final output and continues working on the task
This approach is particularly useful for:

Large data processing: Filter or aggregate tool results before they reach Claude's context
Multi-step workflows: Save tokens and latency by calling tools serially or in a loop without sampling Claude in-between tool calls
Conditional logic: Make decisions based on intermediate tool results


Core concepts
The allowed_callers field
The allowed_callers field specifies which contexts can invoke a tool:

{
  "name": "query_database",
  "description": "Execute a SQL query against the database",
  "input_schema": {
    // ...
  },
  "allowed_callers": ["code_execution_20260120"]
}
Possible values:

["direct"] - Only Claude can call this tool directly (default if omitted)
["code_execution_20260120"] - Only callable from within code execution
["direct", "code_execution_20260120"] - Callable both directly and from code execution
Choose either ["direct"] or ["code_execution_20260120"] for each tool rather than enabling both, as this provides clearer guidance to Claude for how best to use the tool.

The caller field in responses
Every tool use block includes a caller field indicating how it was invoked:

Direct invocation (traditional tool use):

{
  "type": "tool_use",
  "id": "toolu_abc123",
  "name": "query_database",
  "input": { "sql": "<sql>" },
  "caller": { "type": "direct" }
}
Programmatic invocation:

{
  "type": "tool_use",
  "id": "toolu_xyz789",
  "name": "query_database",
  "input": { "sql": "<sql>" },
  "caller": {
    "type": "code_execution_20260120",
    "tool_id": "srvtoolu_abc123"
  }
}
The tool_id references the code execution tool that made the programmatic call.

Container lifecycle
Programmatic tool calling uses the same containers as code execution:

Container creation: A new container is created for each session unless you reuse an existing one
Expiration: Containers expire after approximately 4.5 minutes of inactivity (subject to change)
Container ID: Returned in responses via the container field
Reuse: Pass the container ID to maintain state across requests
When a tool is called programmatically and the container is waiting for your tool result, you must respond before the container expires. Monitor the expires_at field. If the container expires, Claude may treat the tool call as timed out and retry it.

Example workflow
Here's how a complete programmatic tool calling flow works:

Step 1: Initial request
Send a request with code execution and a tool that allows programmatic calling. To enable programmatic calling, add the allowed_callers field to your tool definition.

Provide detailed descriptions of your tool's output format in the tool description. If you specify that the tool returns JSON, Claude will attempt to deserialize and process the result in code. The more detail you provide about the output schema, the better Claude can handle the response programmatically.

Python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=4096,
    messages=[
        {
            "role": "user",
            "content": "Query customer purchase history from the last quarter and identify our top 5 customers by revenue",
        }
    ],
    tools=[
        {"type": "code_execution_20260120", "name": "code_execution"},
        {
            "name": "query_database",
            "description": "Execute a SQL query against the sales database. Returns a list of rows as JSON objects.",
            "input_schema": {
                # ...
            },
            "allowed_callers": ["code_execution_20260120"],
        },
    ],
)
Step 2: API response with tool call
Claude writes code that calls your tool. The API pauses and returns:

{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "I'll query the purchase history and analyze the results."
    },
    {
      "type": "server_tool_use",
      "id": "srvtoolu_abc123",
      "name": "code_execution",
      "input": {
        "code": "results = await query_database('<sql>')\ntop_customers = sorted(results, key=lambda x: x['revenue'], reverse=True)[:5]\nprint(f'Top 5 customers: {top_customers}')"
      }
    },
    {
      "type": "tool_use",
      "id": "toolu_def456",
      "name": "query_database",
      "input": { "sql": "<sql>" },
      "caller": {
        "type": "code_execution_20260120",
        "tool_id": "srvtoolu_abc123"
      }
    }
  ],
  "container": {
    "id": "container_xyz789",
    "expires_at": "2025-01-15T14:30:00Z"
  },
  "stop_reason": "tool_use"
}
Step 3: Provide tool result
Include the full conversation history plus your tool result:

Python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=4096,
    container="container_xyz789",  # Reuse the container
    messages=[
        {
            "role": "user",
            "content": "Query customer purchase history from the last quarter and identify our top 5 customers by revenue",
        },
        {
            "role": "assistant",
            "content": [
                {
                    "type": "text",
                    "text": "I'll query the purchase history and analyze the results.",
                },
                {
                    "type": "server_tool_use",
                    "id": "srvtoolu_abc123",
                    "name": "code_execution",
                    "input": {"code": "..."},
                },
                {
                    "type": "tool_use",
                    "id": "toolu_def456",
                    "name": "query_database",
                    "input": {"sql": "<sql>"},
                    "caller": {
                        "type": "code_execution_20260120",
                        "tool_id": "srvtoolu_abc123",
                    },
                },
            ],
        },
        {
            "role": "user",
            "content": [
                {
                    "type": "tool_result",
                    "tool_use_id": "toolu_def456",
                    "content": '[{"customer_id": "C1", "revenue": 45000}, {"customer_id": "C2", "revenue": 38000}, ...]',
                }
            ],
        },
    ],
    tools=[...],
)
Step 4: Next tool call or completion
The code execution continues and processes the results. If additional tool calls are needed, repeat Step 3 until all tool calls are satisfied.

Step 5: Final response
Once the code execution completes, Claude provides the final response:

{
  "content": [
    {
      "type": "code_execution_tool_result",
      "tool_use_id": "srvtoolu_abc123",
      "content": {
        "type": "code_execution_result",
        "stdout": "Top 5 customers by revenue:\n1. Customer C1: $45,000\n2. Customer C2: $38,000\n3. Customer C5: $32,000\n4. Customer C8: $28,500\n5. Customer C3: $24,000",
        "stderr": "",
        "return_code": 0,
        "content": []
      }
    },
    {
      "type": "text",
      "text": "I've analyzed the purchase history from last quarter. Your top 5 customers generated $167,500 in total revenue, with Customer C1 leading at $45,000."
    }
  ],
  "stop_reason": "end_turn"
}
Advanced patterns
Batch processing with loops
Claude can write code that processes multiple items efficiently:

# async wrapper omitted for clarity
regions = ["West", "East", "Central", "North", "South"]
results = {}
for region in regions:
    data = await query_database(f"<sql for {region}>")
    results[region] = sum(row["revenue"] for row in data)

# Process results programmatically
top_region = max(results.items(), key=lambda x: x[1])
print(f"Top region: {top_region[0]} with ${top_region[1]:,} in revenue")
This pattern:

Reduces model round-trips from N (one per region) to 1
Processes large result sets programmatically before returning to Claude
Saves tokens by only returning aggregated conclusions instead of raw data
Early termination
Claude can stop processing as soon as success criteria are met:

# async wrapper omitted for clarity
endpoints = ["us-east", "eu-west", "apac"]
for endpoint in endpoints:
    status = await check_health(endpoint)
    if status == "healthy":
        print(f"Found healthy endpoint: {endpoint}")
        break  # Stop early, don't check remaining
Conditional tool selection
# async wrapper omitted for clarity
file_info = await get_file_info(path)
if file_info["size"] < 10000:
    content = await read_full_file(path)
else:
    content = await read_file_summary(path)
print(content)
Data filtering
# async wrapper omitted for clarity
logs = await fetch_logs(server_id)
errors = [log for log in logs if "ERROR" in log]
print(f"Found {len(errors)} errors")
for error in errors[-10:]:  # Only return last 10 errors
    print(error)
Response format
Programmatic tool call
When code execution calls a tool:

{
  "type": "tool_use",
  "id": "toolu_abc123",
  "name": "query_database",
  "input": { "sql": "<sql>" },
  "caller": {
    "type": "code_execution_20260120",
    "tool_id": "srvtoolu_xyz789"
  }
}
Tool result handling
Your tool result is passed back to the running code:

{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_abc123",
      "content": "[{\"customer_id\": \"C1\", \"revenue\": 45000, \"orders\": 23}, {\"customer_id\": \"C2\", \"revenue\": 38000, \"orders\": 18}, ...]"
    }
  ]
}
Code execution completion
When all tool calls are satisfied and code completes:

{
  "type": "code_execution_tool_result",
  "tool_use_id": "srvtoolu_xyz789",
  "content": {
    "type": "code_execution_result",
    "stdout": "Analysis complete. Top 5 customers identified from 847 total records.",
    "stderr": "",
    "return_code": 0,
    "content": []
  }
}
