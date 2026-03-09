# Glue Job Retry Automation Documentation

Welcome to the documentation for the **Failure Retry Automation for AWS Glue** project. This automated system intelligently detects, analyzes, and handles AWS Glue job failures using EventBridge, Lambda, and Amazon Bedrock.

## 📚 Documentation Overview

This documentation is organized into the following sections to help you understand and implement the solution:

### [Architecture](architecture.md)
Explore the complete architecture of the solution, including:
- High-level system design
- Component interaction flow
- Data flow diagrams
- Infrastructure components

### [EventBridge Rule](eventbridge-rule.md)
Learn how EventBridge captures Glue job failures:
- Event pattern configuration
- Rule definition and triggers
- Event payload structure
- Integration with Lambda

### [Lambda Function](lambda-function.md)
Understand the Lambda processing logic:
- Event processing workflow
- Failure analysis engine
- Retry decision logic
- State management with DynamoDB

### [Bedrock Integration](bedrock-integration.md)
Discover how Amazon Bedrock enhances failure analysis:
- LLM-based error classification
- Prompt engineering approach
- Pattern matching vs. LLM analysis
- Confidence scoring

---

## 🎯 What This System Does

The Glue Job Retry Automation system provides an intelligent, automated approach to handling AWS Glue job failures:

### 1. **Automatic Detection**
When an AWS Glue job fails, EventBridge immediately captures the failure event and triggers the analysis workflow.

### 2. **Intelligent Analysis**
The system uses a two-tier approach to classify failures:
- **Pattern Matching**: Fast classification using predefined error patterns
- **LLM Analysis**: Amazon Bedrock analyzes complex errors when patterns don't match

### 3. **Smart Retry Logic**
Based on the analysis, the system decides whether to:
- **Automatically Retry**: For transient failures (network issues, timeouts)
- **Notify Team**: For persistent issues requiring manual intervention

### 4. **State Tracking**
DynamoDB tracks retry attempts to prevent infinite loops and maintain visibility into failure history.

### 5. **Team Notifications**
When a job cannot be automatically retried, the team receives detailed notifications via Webex Teams with:
- Error details and classification
- Failure reasoning
- Recommended actions
- Direct links to CloudWatch logs

---

## 🚀 Key Features

- ✅ **Zero Manual Intervention**: Automatically handles transient failures
- 🧠 **AI-Powered Analysis**: Uses Amazon Bedrock for intelligent error classification
- 🔄 **Configurable Retry Limits**: Prevents infinite retry loops
- 📊 **Complete Observability**: CloudWatch metrics and alarms
- 🔐 **Secure by Design**: IAM roles with least privilege access
- 🏗️ **Infrastructure as Code**: Complete Terraform deployment
- 🔧 **Highly Configurable**: Easy customization of error patterns and retry behavior

---

## 📋 Quick Links

- [Main README](../README.md) - Complete use case overview and setup guide
- [Architecture Details](architecture.md) - Deep dive into system design
- [EventBridge Configuration](eventbridge-rule.md) - Event capture setup
- [Lambda Function Guide](lambda-function.md) - Processing logic details
- [Bedrock Integration](bedrock-integration.md) - AI analysis implementation

---

## 🎓 Getting Started

If you're new to this project:

1. Start with the [README](../README.md) to understand the complete use case
2. Review the [Architecture](architecture.md) to see how components interact
3. Explore individual component documentation for implementation details
4. Follow the deployment guide in the README to set up your own instance

---
