# EventBridge Rule Configuration

This page explains how Amazon EventBridge is configured to detect AWS Glue job failures and trigger the automated retry workflow.

---

## 🎯 Overview

Amazon EventBridge serves as the event-driven trigger mechanism for the entire automation system. It monitors AWS Glue job state changes and immediately invokes the Lambda analysis function when a job fails.

### Key Responsibilities

1. **Real-time Monitoring**: Continuously monitors AWS Glue job state changes
2. **Event Filtering**: Captures only FAILED state events
3. **Immediate Triggering**: Invokes Lambda function with minimal latency
4. **Event Enrichment**: Passes complete failure context to Lambda

---

## 🔧 EventBridge Rule Definition

The EventBridge rule is defined in Terraform as follows:

```hcl
resource "aws_cloudwatch_event_rule" "glue_failure" {
  name        = "glue-job-failure-trigger"
  description = "Trigger Lambda when Glue job fails"
  
  event_pattern = jsonencode({
    source      = ["aws.glue"]
    detail-type = ["Glue Job State Change"]
    detail = {
      state = ["FAILED"]
    }
  })
  
  tags = {
    Name = "glue-job-failure-trigger"
  }
}
```

### Configuration Breakdown

| Property | Value | Purpose |
|----------|-------|---------|
| **name** | `glue-job-failure-trigger` | Unique identifier for the rule |
| **source** | `aws.glue` | Filters events from AWS Glue service |
| **detail-type** | `Glue Job State Change` | Specific event type for job state changes |
| **state** | `FAILED` | Only captures failed job states |

---

## 📨 Event Pattern Explained

The event pattern is the heart of the EventBridge rule. It defines exactly which events should trigger the Lambda function.

### Event Pattern Structure

```json
{
  "source": ["aws.glue"],
  "detail-type": ["Glue Job State Change"],
  "detail": {
    "state": ["FAILED"]
  }
}
```

### Why This Pattern?

- **`source: aws.glue`**: Ensures we only process events from AWS Glue service, filtering out unrelated events
- **`detail-type: Glue Job State Change`**: Narrows down to job state change events specifically
- **`state: FAILED`**: Only triggers on failed jobs, ignoring RUNNING, SUCCEEDED, STOPPED, or TIMEOUT states

### Alternative Patterns

You can customize the pattern to capture additional states if needed:

```json
{
  "source": ["aws.glue"],
  "detail-type": ["Glue Job State Change"],
  "detail": {
    "state": ["FAILED", "TIMEOUT"]
  }
}
```

---

## 🎯 EventBridge Target Configuration

The rule is connected to the Lambda function through a target configuration:

```hcl
resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.glue_failure.name
  target_id = "GlueFailureAnalysisLambda"
  arn       = aws_lambda_function.analysis_agent.arn
}
```

### Lambda Permission

EventBridge needs explicit permission to invoke the Lambda function:

```hcl
resource "aws_lambda_permission" "allow_eventbridge" {
  statement_id  = "AllowExecutionFromEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.analysis_agent.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.glue_failure.arn
}
```

---

## 📋 Event Payload Structure

When a Glue job fails, EventBridge delivers a structured event payload to the Lambda function:

```json
{
  "version": "0",
  "id": "c7e1f0d2-1234-5678-abcd-ef1234567890",
  "detail-type": "Glue Job State Change",
  "source": "aws.glue",
  "account": "123456789012",
  "time": "2024-01-15T10:30:45Z",
  "region": "us-east-1",
  "resources": [],
  "detail": {
    "jobName": "my-data-processing-job",
    "jobRunId": "jr_1234abcd5678ef90",
    "state": "FAILED",
    "message": "Connection timeout while reading from S3",
    "startedOn": 1705318200000,
    "completedOn": 1705318245000
  }
}
```

### Key Fields

- **`detail.jobName`**: Name of the failed Glue job
- **`detail.jobRunId`**: Unique identifier for this job run
- **`detail.state`**: Job state (always "FAILED" in our case)
- **`detail.message`**: Error message from Glue
- **`detail.startedOn`**: Timestamp when job started (epoch milliseconds)
- **`detail.completedOn`**: Timestamp when job failed (epoch milliseconds)

---

## 🔍 How EventBridge Captures Failures

### Step-by-Step Process

1. **Job Execution**: AWS Glue job runs and encounters an error
2. **State Change**: Glue service updates job state to "FAILED"
3. **Event Generation**: AWS automatically generates a state change event
4. **Pattern Matching**: EventBridge evaluates the event against all active rules
5. **Rule Match**: Our rule matches because:
   - Source is `aws.glue` ✓
   - Detail-type is `Glue Job State Change` ✓
   - State is `FAILED` ✓
6. **Target Invocation**: EventBridge invokes the Lambda function with the event payload
7. **Lambda Processing**: Lambda receives event and begins analysis

### Latency Considerations

- **Typical Latency**: 1-3 seconds from failure to Lambda invocation
- **Event Delivery**: EventBridge provides at-least-once delivery guarantee
- **Retry Behavior**: EventBridge automatically retries failed Lambda invocations

---

## 🎓 Best Practices

1. **Specific Event Patterns**: Use precise patterns to avoid unnecessary Lambda invocations
2. **Enable Rule Only When Needed**: Disable during maintenance windows to prevent false alerts
3. **Monitor Rule Metrics**: Track invocation counts and failures in CloudWatch
4. **Test Event Patterns**: Use EventBridge's test event feature to validate patterns

---

[← Back to Documentation Home](index.md) | [Next: Lambda Function →](lambda-function.md)
