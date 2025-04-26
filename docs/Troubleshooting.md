# Troubleshooting

## Metadata
- **url**: https://langchain-ai.github.io/langgraph/troubleshooting/errors/?q=

## Overview
This guide provides solutions for common errors encountered when building applications with LangGraph. Each error is identified by a specific `lc_error_code` property that's included when the error is thrown in code.

## Common Errors

### Core LangGraph Errors
- **GRAPH_RECURSION_LIMIT**: Occurs when a graph exceeds the maximum allowed recursive calls
- **INVALID_CONCURRENT_GRAPH_UPDATE**: Triggered when multiple processes attempt to update a graph state simultaneously
- **INVALID_GRAPH_NODE_RETURN_VALUE**: Happens when a node function returns a value that doesn't match the expected state schema
- **MULTIPLE_SUBGRAPHS**: Occurs when there's an issue with nested subgraph configurations
- **INVALID_CHAT_HISTORY**: Thrown when the chat history format is incompatible with the expected structure

### LangGraph Platform Errors
- **INVALID_LICENSE**: Occurs when there are issues with the LangGraph Platform license validation

## Troubleshooting Steps

For each error type, check the following:

1. **GRAPH_RECURSION_LIMIT**:
   - Review your graph for unintended infinite loops
   - Consider implementing a termination condition
   - Increase the recursion limit if needed (but be cautious)

2. **INVALID_CONCURRENT_GRAPH_UPDATE**:
   - Implement proper synchronization mechanisms
   - Ensure only one process updates the graph state at a time
   - Consider using locks or other concurrency control patterns

3. **INVALID_GRAPH_NODE_RETURN_VALUE**:
   - Verify that node functions return values matching the state schema
   - Check for typing issues or missing required fields
   - Ensure any custom state types are properly defined

4. **MULTIPLE_SUBGRAPHS**:
   - Inspect your subgraph configurations for conflicts
   - Validate that subgraph names are unique
   - Check that parent-child graph relationships are correctly defined

5. **INVALID_CHAT_HISTORY**:
   - Ensure the chat history format matches the expected structure
   - Verify that message objects have the required fields
   - Check for any serialization/deserialization issues

6. **INVALID_LICENSE**:
   - Verify your license key is valid and properly configured
   - Check that your license hasn't expired
   - Ensure you're using features covered by your license tier
