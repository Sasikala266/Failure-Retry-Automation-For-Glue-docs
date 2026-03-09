# Lambda Function Processing Logic

This page provides a detailed explanation of how the Lambda function processes Glue job failures, analyzes errors, and makes retry decisions.

---

## 🎯 Overview

The Lambda function (`glue-failure-analysis-agent`) is the brain of the automation system. It orchestrates the entire workflow from receiving failure events to executing retry decisions.

### Core Responsibilities

1. **Event Processing**: Parse and validate EventBridge events
2. **Log Retrieval**: Fetch error logs from CloudWatch
3. **State Management**: Track retry attempts in DynamoDB
4. **Failure Analysis**: Classify errors using patterns and LLM
5. **Decision Making**: Determine whether to retry or notify
6. **Action Execution**: Start job retries or send notifications

---

## 🔄 Processing Workflow

<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>mermaid.initialize({startOnLoad:true});</script>

<div class="mermaid">
graph TD
    A[EventBridge Invokes Lambda] --> B[Parse Event]
    B --> C[Validate Event Structure]
    C --> D{Valid Event?}
    D -->|No| E[Return Error Response]
    D -->|Yes| F[Fetch CloudWatch Logs]
    F --> G[Get Job Details from Glue API]
    G --> H[Check Retry Count in DynamoDB]
    H --> I[Analyze Failure]
    I --> J{Pattern Match?}
    J -->|Yes| K[Use Pattern Classification]
    J -->|No| L[Use Bedrock LLM Analysis]
    K --> M[Make Retry Decision]
    L --> M
    M --> N{Should Retry?}
    N -->|Yes| O[Increment Retry Count]
    O --> P[Start New Job Run]
    P --> Q[Return Success Response]
    N -->|No| R[Send Webex Notification]
</div>
```

---

## 📝 Main Handler Function

The entry point for all Lambda invocations:

```python
def lambda_handler(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
    """
    Main Lambda handler for processing Glue job failure events.
    
    Args:
        event: EventBridge event containing Glue job failure details
        context: Lambda context object
        
    Returns:
        Response dictionary with processing status
    """
    try:
        logger.info(f"Received event: {json.dumps(event)}")
        
        # Parse EventBridge event
        failure_context = parse_event(event)
        
        if not failure_context:
            return {'statusCode': 400, 'body': 'Invalid event format'}
        
        # Analyze the failure
        analysis_result = analyzer.analyze(failure_context)
        
        # Make retry decision
        decision = make_retry_decision(failure_context, analysis_result)
        
        # Execute decision
        execute_decision(failure_context, analysis_result, decision)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'job_name': failure_context.job_name,
                'classification': analysis_result.classification,
                'action': decision.action
            })
        }
    except Exception as e:
        logger.error(f"Error processing event: {str(e)}", exc_info=True)
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}
```

---

## 🔍 Event Parsing

### Parse Event Function

Extracts failure details from the EventBridge event:

```python
def parse_event(event: Dict[str, Any]) -> FailureContext:
    """Parse EventBridge event into FailureContext."""
    detail = event.get('detail', {})
    
    # Extract basic information
    job_name = detail.get('jobName', '')
    job_run_id = detail.get('jobRunId', '')
    state = detail.get('state', '')
    error_message = detail.get('message', 'No error message provided')
    
    # Validate this is a failure event
    if state != 'FAILED':
        return None
    
    # Get job run details from Glue API
    job_run = glue_client.get_job_run(job_name, job_run_id)
    job_parameters = job_run.get('Arguments', {})
    
    # Retrieve error logs from CloudWatch
    log_group = f"/aws-glue/jobs/{job_name}"
    error_logs = cloudwatch_client.get_log_events(
        log_group=log_group,
        log_stream=job_run_id,
        limit=100
    )
    
    # Determine original run ID (for retry tracking)
    original_run_id = job_parameters.get('--original-run-id', job_run_id)
    
    # Get current retry count
    retry_count = retry_tracker.get_retry_count(job_name, original_run_id)
    
    # Create FailureContext
    return FailureContext(
        job_name=job_name,
        job_run_id=job_run_id,
        original_run_id=original_run_id,
        failure_time=datetime.utcnow(),
        error_message=error_message,
        error_logs=error_logs,
        job_parameters=job_parameters,
        retry_count=retry_count
    )
```

### FailureContext Data Model

```python
@dataclass
class FailureContext:
    job_name: str                    # Name of the failed job
    job_run_id: str                  # Current job run ID
    original_run_id: str             # Original run ID (for tracking retries)
    failure_time: datetime           # When the failure occurred
    error_message: str               # Error message from Glue
    error_logs: str                  # Full error logs from CloudWatch
    job_parameters: Dict[str, str]   # Job arguments/parameters
    retry_count: int                 # Number of retries so far
```

---

## 🧠 Failure Analysis

### Two-Tier Analysis Approach

The system uses a hierarchical approach to error classification:

#### 1. Pattern-Based Analysis (Fast Path)

```python
# Check against predefined patterns
pattern_match = match_error_pattern(failure_context.error_message)

if pattern_match:
    # Use pattern-based classification
    is_retriable = pattern_match == "retriable"
    classification = "transient" if is_retriable else "code_issue"
    confidence = 0.9  # High confidence for pattern matches
```

**Predefined Patterns**:

Retriable patterns:
- Connection timeout
- Network error
- SocketTimeoutException
- Service unavailable
- Throttling errors

Non-retriable patterns:
- Schema mismatch
- Column not found
- SyntaxError
- Permission denied
- ValidationException

#### 2. LLM-Based Analysis (Complex Errors)

When no pattern matches, the system uses Amazon Bedrock:

```python
# Invoke Bedrock for analysis
llm_result = bedrock_client.analyze_error(
    error_message=failure_context.error_message,
    error_logs=failure_context.error_logs
)

# Parse LLM response
is_retriable = llm_result.get("classification") == "RETRIABLE"
confidence = llm_result.get("confidence", 0.5)
reasoning = llm_result.get("reasoning", "")
```

[Learn more about Bedrock integration →](bedrock-integration.md)

---

## 🎯 Retry Decision Logic

### Decision Function

```python
def make_retry_decision(failure_context: FailureContext, 
                       analysis_result) -> RetryDecision:
    """Make decision about whether to retry the job."""
    
    # Check if max retries exceeded
    if not retry_tracker.can_retry(failure_context.job_name, 
                                   failure_context.original_run_id):
        return RetryDecision(
            should_retry=False,
            reason=f"Maximum retry count ({MAX_RETRY_COUNT}) exceeded",
            action="notify"
        )
    
    # Check if failure is retriable
    if analysis_result.is_retriable:
        return RetryDecision(
            should_retry=True,
            reason=f"Failure classified as {analysis_result.classification}",
            action="retry"
        )
    else:
        return RetryDecision(
            should_retry=False,
            reason=f"Failure classified as {analysis_result.classification}",
            action="notify"
        )
```

### Decision Criteria

The system decides to **RETRY** if:
1. ✅ Failure is classified as retriable (transient issue)
2. ✅ Retry count is below maximum limit (default: 3)

The system decides to **NOTIFY** if:
1. ❌ Failure is non-retriable (permanent issue)
2. ❌ Maximum retry count exceeded

---

## ⚡ Decision Execution

### Retry Action

```python
if decision.action == "retry":
    # Increment retry count in DynamoDB
    new_count = retry_tracker.increment_retry_count(
        job_name=failure_context.job_name,
        original_run_id=failure_context.original_run_id,
        current_run_id=failure_context.job_run_id,
        failure_reason=failure_context.error_message
    )
    
    # Prepare job arguments with original run ID
    job_arguments = failure_context.job_parameters.copy()
    job_arguments['--original-run-id'] = failure_context.original_run_id
    
    # Start job retry
    new_run_id = glue_client.start_job_run(
        job_name=failure_context.job_name,
        arguments=job_arguments
    )
    
    logger.info(f"Successfully started retry: {new_run_id} (attempt {new_count})")
```

### Notify Action

```python
elif decision.action == "notify":
    # Send notification to Webex
    retry_exhausted = failure_context.retry_count >= MAX_RETRY_COUNT
    success = webex_notifier.send_notification(
        failure_context, 
        analysis_result, 
        retry_exhausted=retry_exhausted
    )
    
    if success:
        logger.info("Notification sent successfully")
```

---

## 🔐 AWS Client Wrappers

The Lambda function uses custom AWS client wrappers with built-in retry logic:

### CloudWatch Logs Client
- Fetches error logs from Glue job log streams
- Implements exponential backoff retry
- Handles throttling and transient errors

### Glue Client
- Retrieves job run details
- Starts new job runs for retries
- Passes original run ID for tracking

### Bedrock Client
- Invokes Claude model for error analysis
- Handles model-specific request/response format
- Falls back to conservative classification on errors

---

## 📊 State Management with DynamoDB

The Lambda function tracks retry state in DynamoDB to:
- Prevent infinite retry loops
- Maintain failure history
- Enable retry analytics

[Learn more in the Architecture documentation →](architecture.md#3-dynamodb-table)

---

[← Back: EventBridge Rule](eventbridge-rule.md) | [Documentation Home](index.md) | [Next: Bedrock Integration →](bedrock-integration.md)
