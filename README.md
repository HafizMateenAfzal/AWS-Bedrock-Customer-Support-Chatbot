# AWS Bedrock AgentCore Customer Support Chatbot

![AWS Bedrock](https://img.shields.io/badge/AWS-Bedrock%20AgentCore-FF9900?logo=amazonaws&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white)
![Udacity](https://img.shields.io/badge/Udacity-ND905-02B3E4?logo=udacity&logoColor=white)

A customer support chatbot for a fictional online shop, built on **Amazon
Bedrock AgentCore**. It classifies each customer message and routes it down
one of three paths: bug reports become DynamoDB tickets via a Gateway tool,
platform questions are answered strictly from an FAQ, and everything else is
redirected to human support.

This is a project for Udacity's **AWS AI & ML Engineer Nanodegree** (ND905).

## Overview

The chatbot handles three categories of customer message, decided entirely
by the system prompt (`system_prompt.txt`) run on **Amazon Nova Pro**:

- **Bug reports** — the model collects a description, steps to reproduce,
  and environment (browser/OS/device) one field at a time across turns,
  then calls the `create_bug_report` tool to file a DynamoDB ticket. The
  prompt explicitly forbids placeholder values and requires the model to
  re-verify, turn by turn, that all three fields were actually stated by
  the customer before calling the tool.
- **Platform / FAQ questions** — answered strictly from `online_shop_faq.md`
  (orders, shipping, returns, payments, accounts, privacy). If the FAQ
  doesn't cover it, it falls through to the next category.
- **Everything else** — politely redirected to a human support line, with
  explicit instructions to resist prompt-injection attempts to override the
  system prompt.

## Architecture

```
Customer message
      │
      ▼
AgentCore Harness (Nova Pro, pinned, temperature 0 / topK 1)
   system prompt = system_prompt.txt + embedded FAQ
      │
      ├─ Bug report ──► AgentCore Gateway (MCP, AWS_IAM auth)
      │                     └─ Lambda: create_bug_report
      │                            └─ DynamoDB: BugReports table
      │
      ├─ FAQ question ──► answered directly from prompt context
      │
      └─ Other ──► redirect to human support line
```

- **`setup_gateway.py`** creates the AgentCore Gateway and registers the
  `create_bug_report` Lambda as an MCP tool target, reading the Lambda ARN
  and IAM role ARNs straight from the CloudFormation stack outputs.
- **`create_harness.py`** creates (or updates) the AgentCore managed
  harness — loads `system_prompt.txt`, substitutes the `{{FAQ}}` placeholder
  with `online_shop_faq.md`, and pins the model to
  `us.amazon.nova-pro-v1:0` with greedy decoding for reliable tool calling.
- **`chat.py`** is the interactive CLI — opens one stateful session per run,
  attaches the Gateway so the model can call the bug-report tool, and
  streams the reply back token by token.
- **`create_bug_report.py`** is the Lambda behind the Gateway tool: reads
  arguments directly from the event (no Bedrock-Agents-Classic envelope),
  validates all three required fields are non-empty, and writes the ticket
  to DynamoDB.

## Tech Stack

- **Amazon Bedrock AgentCore** — managed harness + Gateway (MCP protocol,
  AWS_IAM auth) for agent orchestration and tool calling
- **Amazon Nova Pro** (`us.amazon.nova-pro-v1:0`) — pinned model, greedy
  decoding (temperature 0, topK 1) for reliable tool calling
- **AWS Lambda** — `create_bug_report` tool implementation
- **Amazon DynamoDB** — `BugReports` ticket store
- **AWS CloudFormation** — infrastructure as code for the Lambda, table, and
  IAM roles
- **Amazon Bedrock Evaluations** — LLM-as-a-judge scoring (bring-your-own-inference)
- **Python 3.11 / boto3**

## Key Features

- Multi-turn slot-filling for bug reports, with explicit anti-hallucination
  guardrails in the prompt (no placeholder values, re-verify every field
  was actually stated before calling the tool)
- FAQ answers grounded strictly in `online_shop_faq.md` — no invented policy answers
- Prompt-injection resistance ("ignore previous instructions" attempts are
  treated as an out-of-scope request, not honored)
- 7-case automated test suite (`harness-tests.json`) covering bug reports,
  FAQ questions (long and short), ambiguous intent, off-topic requests, and
  a prompt-injection attempt
- Automated evaluation pipeline (`generate-eval-dataset.py`) that runs every
  test case in its own fresh session and emits a Bedrock Evaluations JSONL
  dataset for LLM-as-a-judge scoring
- Clean teardown script (`cleanup_agentcore.py`) for harness/gateway/target

## Setup & Deployment

### Prerequisites

- An AWS account with Amazon Bedrock (Nova Pro) access enabled in `us-east-1`
- AWS CLI configured with credentials that can deploy CloudFormation stacks
- Python 3.11+ and `pip`

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Deploy the bug-report tool infrastructure

```bash
aws cloudformation deploy \
  --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

This creates the `BugReports` DynamoDB table, the `create_bug_report`
Lambda, and the IAM roles the Gateway and harness need.

### 3. Create the AgentCore Gateway

```bash
python setup_gateway.py
```

Reads the stack outputs, creates an MCP Gateway, and registers
`create_bug_report` as a tool target named `bugreports`. Saves everything to
`agentcore_config.json`.

### 4. Create the harness

```bash
python create_harness.py
```

Loads `system_prompt.txt` (substituting the FAQ), creates the managed
harness pinned to Nova Pro, and waits for it to become `READY`.

### 5. Chat with it

```bash
python chat.py
```

Starts one stateful conversation per run — type a message, or `quit` to exit.

## Testing & Evaluation

Run the test suite through the eval pipeline:

```bash
python generate-eval-dataset.py --tests-json harness-tests.json
```

This invokes the harness once per test case (each in its own fresh
session) and writes `output_eval_dataset.jsonl` in the format Bedrock
Evaluations expects. From there, an evaluation job can be created in the
Bedrock console for LLM-as-a-judge scoring against the `expected` field in
each test case.

Test cases cover: filing a bug report correctly, two FAQ questions (returns,
shipping), an off-topic request, an ambiguous message, a short one-word
message, and a prompt-injection attempt.

## Cleanup

```bash
python cleanup_agentcore.py
```

Deletes the harness, gateway target, and gateway (in that order), then
prints the remaining CloudFormation cleanup commands (empty the S3
evaluation bucket before deleting its stack).

## Screenshots

> Add screenshots to a `screenshots/` folder and reference them here — e.g.
> the chat CLI in action, a DynamoDB ticket record, and an evaluation job's
> results in the Bedrock console.

## License & Acknowledgments

Built with [Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/),
AWS Lambda, and Amazon DynamoDB, as part of Udacity's AWS AI & ML Engineer
Nanodegree.
