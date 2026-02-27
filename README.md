# Programmatic-tool-calling
Programmatic tool calling allows Claude to write code that calls your tools programmatically within a code execution container, rather than requiring round trips through the model for each tool invocation.
This reduces latency for multi-tool workflows and decreases token consumption by allowing Claude to filter or process data before it reaches the model's context window. On agentic search benchmarks like BrowseComp and DeepSearchQA, which test multi-step web research and complex information retrieval, adding programmatic tool calling on top of basic search tools was the key factor that fully unlocked agent performance.

The difference compounds fast in real workflows. Consider checking budget compliance across 20 employees: the traditional approach requires 20 separate model round-trips, pulling thousands of expense line items into the context along the way. With programmatic tool calling, a single script runs all 20 lookups, filters the results, and returns only the employees who exceeded their limits, shrinking what Claude needs to reason over from hundreds of kilobytes down to a handful of lines.

Quick start
Here's a simple example where Claude programmatically queries a database multiple times and aggregates results:

curl https://api.anthropic.com/v1/messages \
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
