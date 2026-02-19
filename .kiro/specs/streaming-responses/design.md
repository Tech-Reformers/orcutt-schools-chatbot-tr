# Design Document: Streaming Responses

## Overview

This design implements streaming responses for the Orcutt Schools chatbot by leveraging AWS Lambda response streaming with the Bedrock Claude API. The system will stream tokens from Claude through Lambda to API Gateway and finally to the React frontend, providing immediate visual feedback to users instead of waiting 10-15 seconds for complete responses.

The implementation uses AWS Lambda's response streaming feature (introduced in April 2023) which allows Lambda functions to stream responses back through API Gateway using a streaming response handler. This approach maintains the existing serverless architecture while enabling real-time token delivery.

### Key Design Decisions

1. **Lambda Response Streaming**: Use AWS Lambda's native response streaming capability with `awslambda.stream_response_handler` decorator instead of WebSockets or SSE, maintaining the existing REST API architecture
2. **Bedrock Streaming API**: Use `invoke_model_with_response_stream` from boto3 to receive tokens from Claude as they're generated
3. **Frontend EventSource**: Use browser's native EventSource API to consume the Lambda streaming response
4. **Backward Compatibility**: Preserve all existing functionality including conversation context, query classification, sources, and metadata

## Architecture

### High-Level Flow

```
User Query → React Frontend → API Gateway → Lambda (Streaming) → Bedrock Claude API
                    ↑                                    ↓
                    └────────── Streaming Tokens ───────┘
```

### Component Interaction

1. **Frontend** submits query to API Gateway endpoint
2. **API Gateway** invokes Lambda function with streaming enabled
3. **Lambda** performs query classification and retrieval (non-streaming)
4. **Lambda** invokes Bedrock with streaming enabled
5. **Lambda** forwards each token chunk to the response stream
6. **API Gateway** forwards stream chunks to Frontend
7. **Frontend** appends each token to the displayed response
8. **Lambda** sends final metadata chunk (sources, response time) after streaming completes
9. **Frontend** displays sources and marks response as complete

### Streaming Protocol

The Lambda function will use AWS Lambda response streaming with a custom event format:

```json
// Token chunk
{"type": "token", "content": "Hello"}

// Completion chunk
{"type": "done", "sources": [...], "metadata": {...}}

// Error chunk
{"type": "error", "message": "Connection failed"}
```

Each chunk is sent as a newline-delimited JSON object, allowing the frontend to parse and process events incrementally.

## Components and Interfaces

### Backend Components

#### 1. Lambda Handler (Streaming)

**File**: `lambda/chatbot/lambda_function.py`

**New Streaming Handler**:
```python
@awslambda.stream_response_handler
def lambda_handler_streaming(event, response_stream, context):
    """
    Streaming Lambda handler that processes chat requests and streams responses.
    
    Args:
        event: API Gateway event containing user query
        response_stream: Lambda response stream for sending chunks
        context: Lambda context object
    
    Returns:
        None (writes to response_stream)
    """
```

**Key Methods**:
- `stream_bedrock_response(query, context, response_stream)`: Invokes Bedrock with streaming and forwards tokens
- `send_chunk(response_stream, chunk_type, data)`: Sends formatted JSON chunk to stream
- `handle_streaming_error(response_stream, error)`: Sends error chunk and closes stream gracefully

#### 2. Bedrock Streaming Client

**Integration Point**: Within `OrcuttChatbot` class

**New Method**:
```python
def generate_response_streaming(self, query: str, context: str, 
                                query_type: str, conversation_context: str, 
                                selected_school: str, response_stream) -> None:
    """
    Generate streaming response using Claude with conversation context.
    
    Args:
        query: User query
        context: Knowledge base context
        query_type: Classification result
        conversation_context: Conversation history
        selected_school: Selected school filter
        response_stream: Lambda response stream
    
    Side Effects:
        Writes token chunks to response_stream
    """
```

**Bedrock API Call**:
```python
response = self.bedrock_client.invoke_model_with_response_stream(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    body=json.dumps(body),
    contentType='application/json'
)

# Process stream
for event in response['body']:
    chunk = json.loads(event['chunk']['bytes'])
    if chunk['type'] == 'content_block_delta':
        token = chunk['delta']['text']
        # Send token to response_stream
```

### Frontend Components

#### 1. Streaming API Service

**File**: `frontend/src/services/apiService.js`

**New Method**:
```javascript
/**
 * Send a chat message with streaming response
 * @param {string} message - User message
 * @param {string} sessionId - Session identifier
 * @param {string} selectedSchool - Selected school (optional)
 * @param {Function} onToken - Callback for each token
 * @param {Function} onComplete - Callback when stream completes
 * @param {Function} onError - Callback for errors
 * @returns {Function} Cleanup function to abort stream
 */
sendMessageStreaming: async (message, sessionId, selectedSchool, 
                             onToken, onComplete, onError) => {
  const url = `${API_BASE_URL}/chat`;
  const eventSource = new EventSource(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message, sessionId, selectedSchool })
  });
  
  eventSource.onmessage = (event) => {
    const chunk = JSON.parse(event.data);
    if (chunk.type === 'token') {
      onToken(chunk.content);
    } else if (chunk.type === 'done') {
      onComplete(chunk);
      eventSource.close();
    }
  };
  
  eventSource.onerror = (error) => {
    onError(error);
    eventSource.close();
  };
  
  return () => eventSource.close();
}
```

#### 2. Chat Hook (Streaming)

**File**: `frontend/src/hooks/useChat.js`

**Modified Method**:
```javascript
const sendMessage = useCallback(async (message) => {
  if (!message.trim() || isLoading || !sessionId) return;
  
  setError(null);
  addMessage('user', message);
  setIsLoading(true);
  
  // Create placeholder for streaming response
  const assistantMessageId = Date.now() + Math.random();
  setMessages(prev => [...prev, {
    id: assistantMessageId,
    role: 'assistant',
    content: '',
    timestamp: new Date(),
    isStreaming: true
  }]);
  
  try {
    const cleanup = await chatService.sendMessageStreaming(
      message,
      sessionId,
      selectedSchool,
      // onToken callback
      (token) => {
        setMessages(prev => prev.map(msg =>
          msg.id === assistantMessageId
            ? { ...msg, content: msg.content + token }
            : msg
        ));
      },
      // onComplete callback
      (data) => {
        setMessages(prev => prev.map(msg =>
          msg.id === assistantMessageId
            ? {
                ...msg,
                isStreaming: false,
                responseTime: data.responseTime,
                queryType: data.queryType,
                sources: data.sources || []
              }
            : msg
        ));
        setSources(data.sources || []);
        setIsLoading(false);
      },
      // onError callback
      (error) => {
        setError(error.message);
        setMessages(prev => prev.map(msg =>
          msg.id === assistantMessageId
            ? { ...msg, isStreaming: false, isError: true }
            : msg
        ));
        setIsLoading(false);
      }
    );
    
    // Store cleanup function for potential cancellation
    return cleanup;
    
  } catch (err) {
    setError(err.message);
    setIsLoading(false);
  }
}, [isLoading, sessionId, selectedSchool]);
```

#### 3. Message Display Component

**File**: `frontend/src/components/MessageBubble.js`

**Streaming Indicator**:
```javascript
const MessageBubble = ({ message }) => {
  return (
    <div className={`message ${message.role}`}>
      <div className="message-content">
        {message.content}
        {message.isStreaming && (
          <span className="streaming-cursor">▊</span>
        )}
      </div>
    </div>
  );
};
```

### Infrastructure Components

#### API Gateway Configuration

**File**: `infrastructure/orcutt_chatbot_stack.py`

**Modified Lambda Integration**:
```python
# Enable streaming for Lambda integration
lambda_integration = apigateway.LambdaIntegration(
    chatbot_lambda,
    proxy=True,
    integration_responses=[
        apigateway.IntegrationResponse(
            status_code="200",
            response_parameters={
                'method.response.header.Content-Type': "'text/event-stream'",
                'method.response.header.Cache-Control': "'no-cache'",
                'method.response.header.Connection': "'keep-alive'"
            }
        )
    ]
)

# Update method response
chat_resource.add_method(
    "POST",
    lambda_integration,
    method_responses=[
        apigateway.MethodResponse(
            status_code="200",
            response_parameters={
                'method.response.header.Content-Type': True,
                'method.response.header.Cache-Control': True,
                'method.response.header.Connection': True
            }
        )
    ]
)
```

**Lambda Function URL** (Alternative Approach):
If API Gateway streaming proves problematic, use Lambda Function URLs which have native streaming support:

```python
# Add Function URL to Lambda
chatbot_lambda_url = chatbot_lambda.add_function_url(
    auth_type=lambda_.FunctionUrlAuthType.NONE,
    cors=lambda_.FunctionUrlCorsOptions(
        allowed_origins=["*"],
        allowed_methods=[lambda_.HttpMethod.POST],
        allowed_headers=["*"]
    ),
    invoke_mode=lambda_.InvokeMode.RESPONSE_STREAM
)
```

## Data Models

### Request Model

```typescript
interface ChatRequest {
  message: string;
  sessionId: string;
  selectedSchool?: string;
}
```

### Streaming Response Chunks

```typescript
// Token chunk
interface TokenChunk {
  type: 'token';
  content: string;
}

// Completion chunk
interface CompletionChunk {
  type: 'done';
  messageId: string;
  responseTime: number;
  queryType: string;
  sources: Source[];
}

// Error chunk
interface ErrorChunk {
  type: 'error';
  message: string;
  partialContent?: string;
}

type StreamChunk = TokenChunk | CompletionChunk | ErrorChunk;
```

### Source Model (Unchanged)

```typescript
interface Source {
  filename: string;
  url: string | null;
  s3Uri: string | null;
}
```

### Message Model (Extended)

```typescript
interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  isStreaming?: boolean;
  isError?: boolean;
  responseTime?: number;
  queryType?: string;
  sources?: Source[];
}
```


## Error Handling

### Backend Error Handling

#### 1. Bedrock Streaming Errors

**Scenario**: Claude API streaming connection fails mid-stream

**Handling**:
```python
try:
    for event in response['body']:
        chunk = json.loads(event['chunk']['bytes'])
        # Process chunk
except Exception as e:
    logger.error(f"Streaming error: {str(e)}")
    send_chunk(response_stream, 'error', {
        'message': 'Streaming interrupted',
        'partialContent': accumulated_content
    })
    # Save partial response to DynamoDB
    save_conversation_to_dynamodb(
        session_id, message, accumulated_content, 
        [], response_time, 'streaming_error'
    )
```

#### 2. Lambda Timeout

**Scenario**: Lambda function times out during streaming

**Handling**:
- Set Lambda timeout to 60 seconds (increased from 30)
- Monitor remaining time using `context.get_remaining_time_in_millis()`
- Send completion chunk with partial content if timeout approaching:

```python
if context.get_remaining_time_in_millis() < 5000:  # 5 seconds remaining
    send_chunk(response_stream, 'done', {
        'messageId': message_id,
        'sources': sources,
        'responseTime': elapsed_time,
        'truncated': True
    })
    break
```

#### 3. Query Classification Errors

**Scenario**: Nova classification fails before streaming begins

**Handling**:
- Fall back to 'knowledge_base' classification
- Log error but continue with streaming
- No impact on user experience

#### 4. Knowledge Base Retrieval Errors

**Scenario**: Bedrock knowledge base query fails

**Handling**:
- Continue with streaming using empty context
- Claude will respond based on conversation history only
- Log error for monitoring

### Frontend Error Handling

#### 1. Connection Interruption

**Scenario**: Network connection drops during streaming

**Handling**:
```javascript
eventSource.onerror = (error) => {
  // Display partial response with error indicator
  setMessages(prev => prev.map(msg =>
    msg.id === assistantMessageId
      ? {
          ...msg,
          isStreaming: false,
          isError: true,
          errorMessage: 'Connection interrupted'
        }
      : msg
  ));
  
  // Provide retry option
  setRetryAvailable(true);
};
```

#### 2. Timeout Waiting for First Token

**Scenario**: No tokens received within 5 seconds

**Handling**:
```javascript
const timeoutId = setTimeout(() => {
  if (!firstTokenReceived) {
    setError('Response timeout - the server is taking longer than expected');
    // Keep waiting but show warning
  }
}, 5000);

// Clear timeout when first token arrives
onToken = (token) => {
  clearTimeout(timeoutId);
  firstTokenReceived = true;
  // Append token
};
```

#### 3. Malformed Chunk

**Scenario**: Received chunk is not valid JSON

**Handling**:
```javascript
eventSource.onmessage = (event) => {
  try {
    const chunk = JSON.parse(event.data);
    // Process chunk
  } catch (e) {
    console.error('Malformed chunk:', event.data);
    // Continue streaming, ignore bad chunk
  }
};
```

#### 4. Empty Response

**Scenario**: Stream completes with no tokens

**Handling**:
```javascript
onComplete = (data) => {
  if (currentContent.length === 0) {
    setMessages(prev => prev.map(msg =>
      msg.id === assistantMessageId
        ? {
            ...msg,
            content: 'I apologize, but I was unable to generate a response. Please try again.',
            isError: true
          }
        : msg
    ));
  }
};
```

### Error Recovery

#### Retry Mechanism

**User-Initiated Retry**:
```javascript
const retryMessage = useCallback((messageId) => {
  const originalMessage = messages.find(m => m.id === messageId - 1);
  if (originalMessage && originalMessage.role === 'user') {
    // Remove failed response
    setMessages(prev => prev.filter(m => m.id !== messageId));
    // Resend original message
    sendMessage(originalMessage.content);
  }
}, [messages, sendMessage]);
```

#### Automatic Fallback

If streaming fails repeatedly, fall back to non-streaming mode:
```javascript
const [streamingEnabled, setStreamingEnabled] = useState(true);
const [failureCount, setFailureCount] = useState(0);

if (failureCount >= 3) {
  setStreamingEnabled(false);
  // Use original non-streaming API
}
```

### Logging and Monitoring

**Backend Logging**:
```python
logger.info(f"Streaming started: session={session_id}, query_type={query_type}")
logger.info(f"First token sent: latency={first_token_latency}ms")
logger.info(f"Streaming completed: tokens={token_count}, time={total_time}s")
logger.error(f"Streaming error: {error_type}, session={session_id}")
```

**Frontend Logging**:
```javascript
console.log('Stream started:', { sessionId, messageId });
console.log('First token received:', { latency: Date.now() - startTime });
console.log('Stream completed:', { tokenCount, totalTime });
console.error('Stream error:', { error, sessionId, messageId });
```

## Testing Strategy

### Unit Testing

#### Backend Unit Tests

**File**: `lambda/chatbot/test_streaming.py`

**Test Cases**:
1. **Test Bedrock streaming integration**
   - Mock `invoke_model_with_response_stream` response
   - Verify tokens are extracted correctly from chunks
   - Verify sources are parsed from final response

2. **Test chunk formatting**
   - Verify token chunks have correct format
   - Verify completion chunks include all metadata
   - Verify error chunks include error message

3. **Test error handling**
   - Simulate Bedrock API failure mid-stream
   - Verify error chunk is sent
   - Verify partial content is saved to DynamoDB

4. **Test timeout handling**
   - Mock context with low remaining time
   - Verify early completion chunk is sent
   - Verify truncated flag is set

5. **Test conversation context preservation**
   - Verify conversation history is included in prompt
   - Verify streaming doesn't affect context retrieval

#### Frontend Unit Tests

**File**: `frontend/src/services/apiService.test.js`

**Test Cases**:
1. **Test EventSource connection**
   - Mock EventSource API
   - Verify connection is established with correct URL
   - Verify headers are set correctly

2. **Test token callback**
   - Send mock token chunks
   - Verify onToken callback is invoked
   - Verify tokens are passed correctly

3. **Test completion callback**
   - Send mock completion chunk
   - Verify onComplete callback is invoked
   - Verify metadata is passed correctly

4. **Test error callback**
   - Simulate connection error
   - Verify onError callback is invoked
   - Verify EventSource is closed

5. **Test cleanup function**
   - Call cleanup function
   - Verify EventSource is closed
   - Verify no memory leaks

**File**: `frontend/src/hooks/useChat.test.js`

**Test Cases**:
1. **Test message streaming**
   - Send message with streaming
   - Verify placeholder message is created
   - Verify content is appended incrementally

2. **Test streaming completion**
   - Complete stream with metadata
   - Verify message is marked as complete
   - Verify sources are updated

3. **Test streaming error**
   - Simulate streaming error
   - Verify error message is displayed
   - Verify retry option is available

4. **Test loading state**
   - Verify loading state during streaming
   - Verify loading state cleared on completion
   - Verify loading state cleared on error

### Integration Testing

**Test Scenarios**:

1. **End-to-end streaming flow**
   - Submit query from frontend
   - Verify tokens appear incrementally
   - Verify sources appear after completion
   - Verify response time is displayed

2. **Conversation context with streaming**
   - Send multiple messages in sequence
   - Verify context is preserved across streaming responses
   - Verify follow-up questions work correctly

3. **Query classification with streaming**
   - Test greeting query (non-streaming response)
   - Test farewell query (non-streaming response)
   - Test knowledge base query (streaming response)

4. **School-specific queries with streaming**
   - Select specific school
   - Submit school-specific query
   - Verify streaming response includes school-filtered sources

5. **Error recovery**
   - Simulate network interruption
   - Verify partial response is displayed
   - Verify retry button works

6. **Performance testing**
   - Measure time to first token
   - Verify first token arrives within 2 seconds
   - Measure total streaming time
   - Compare to non-streaming baseline

### Property-Based Testing

Property-based tests will be defined after completing the prework analysis in the next section.


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Backend Properties

**Property 1: Streaming API Invocation**
*For any* user query, when the Backend processes it, the Backend should invoke the Bedrock API using `invoke_model_with_response_stream` rather than the non-streaming `invoke_model` method.
**Validates: Requirements 1.1, 6.1**

**Property 2: Incremental Token Forwarding**
*For any* streaming response from Bedrock, each token chunk should be forwarded to the response stream immediately upon receipt, without waiting for subsequent chunks or the complete response.
**Validates: Requirements 1.5, 6.3, 6.4, 7.2**

**Property 3: Query Classification Before Streaming**
*For any* user query, query classification should be performed and completed before invoking the Bedrock streaming API.
**Validates: Requirements 2.2**

**Property 4: Conversation Context Preservation**
*For any* sequence of messages in a session, when generating a streaming response, the conversation context passed to Claude should include all previous messages from that session in chronological order.
**Validates: Requirements 2.1**

**Property 5: Model Parameter Consistency**
*For any* streaming API invocation, the Claude model parameters (temperature, top_p, max_tokens, model ID) should match the parameters used in the current non-streaming implementation.
**Validates: Requirements 6.2**

**Property 6: Completion Signal**
*For any* streaming response, when the Bedrock stream completes, a completion chunk containing metadata (sources, response time, message ID) should be sent to the response stream.
**Validates: Requirements 5.1, 6.5, 9.1, 9.5**

**Property 7: Error Chunk on Failure**
*For any* streaming response, if an error occurs during streaming, an error chunk containing the error message and any accumulated partial content should be sent to the response stream.
**Validates: Requirements 3.5**

### Frontend Properties

**Property 8: Token Appending**
*For any* sequence of tokens received from the stream, each token should be appended to the message content in the order received, without reordering or dropping tokens.
**Validates: Requirements 1.3, 8.3**

**Property 9: Connection Establishment**
*For any* user query submission, a streaming connection to the API Gateway should be established before the query is sent.
**Validates: Requirements 7.1, 8.1**

**Property 10: Chunk Parsing**
*For any* chunk received from the stream, the Frontend should successfully parse it as JSON and extract the chunk type and content.
**Validates: Requirements 8.2**

**Property 11: Loading Indicator Lifecycle**
*For any* query submission, a loading indicator should be displayed immediately upon submission and should be removed when the first token arrives or an error occurs.
**Validates: Requirements 4.1, 4.2**

**Property 12: Source Display Timing**
*For any* streaming response, sources should not be displayed while `isStreaming` is true, and should only be displayed after the completion chunk is received.
**Validates: Requirements 5.3**

**Property 13: Source Format Consistency**
*For any* list of sources, the display format in streaming mode should match the display format in non-streaming mode (same fields, same ordering, same styling).
**Validates: Requirements 2.3, 5.4**

**Property 14: Metadata Preservation**
*For any* completed streaming response, all metadata fields (responseTime, queryType, messageId, sources) should be present and should match the format of non-streaming responses.
**Validates: Requirements 9.2, 9.3, 9.4**

**Property 15: Error Handling with Retry**
*For any* streaming error, the Frontend should display an error indicator and provide a retry option to the user.
**Validates: Requirements 3.3, 8.4**

**Property 16: Connection Cleanup**
*For any* streaming connection, when the stream completes (either successfully or with an error), the connection should be closed and the message should be marked as not streaming.
**Validates: Requirements 7.3, 8.5**

### Integration Properties

**Property 17: End-to-End Token Flow**
*For any* user query that triggers a knowledge base response, tokens generated by Claude should flow from Bedrock → Lambda → API Gateway → Frontend without buffering the complete response at any stage.
**Validates: Requirements 7.2**

**Property 18: Connection Timeout Configuration**
*For any* Lambda function configuration, the timeout value should be at least 30 seconds to support streaming responses.
**Validates: Requirements 7.4**

### Edge Cases and Examples

**Example 1: Bedrock Streaming Failure**
When the Bedrock streaming API throws an exception mid-stream, the Backend should send an error chunk with the partial content accumulated before the failure.
**Validates: Requirements 3.1**

**Example 2: Connection Interruption**
When the EventSource connection is interrupted (network failure), the Frontend should display the partial response received so far with an error indicator.
**Validates: Requirements 3.2**

**Example 3: Backend Timeout**
When the Lambda function is approaching timeout (less than 5 seconds remaining), the Backend should send a completion chunk with a truncated flag and the partial response.
**Validates: Requirements 3.4**

**Example 4: First Token Timeout Warning**
When no tokens are received within 5 seconds of query submission, the Frontend should display a timeout warning while continuing to wait for tokens.
**Validates: Requirements 4.4**

**Example 5: Empty Sources**
When a streaming response completes with an empty sources array, the Frontend should display the response without a sources section.
**Validates: Requirements 5.5**

**Example 6: Connection Interruption Notification**
When the streaming connection is interrupted, both the Backend (via Lambda context) and Frontend (via EventSource error event) should be notified of the interruption.
**Validates: Requirements 7.5**

