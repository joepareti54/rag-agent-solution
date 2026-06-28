# rag-agent-solution

Design for a multi-agent orchestration layer over a serverless RAG system on AWS.

## Status

**Design phase.** Architecture and technology choices are documented in
[PLAN.md](PLAN.md). Implementation has not started.

## What this is

An existing serverless RAG pipeline on AWS answers questions over a
private corpus. This project extends it into a multi-agent system: a
manager agent (built on smolagents) reasons about each question and
decides which workers to invoke — the existing RAG endpoint, a
Bedrock-hosted model, or both.

Because agent reasoning loops can take minutes, the public API is
asynchronous (submit a job, poll for the result).

Full architecture, component breakdown, and rationale: **[PLAN.md](PLAN.md)**.

## Repository contents

| File | Purpose |
|---|---|
| [`PLAN.md`](PLAN.md) | Architecture and design decisions. |
| `specs.txt` | Original requirements. |
| `Technical Guide_ Building a Serverless RAG System on AWS with FAISS and Bedrock (7).pdf` | Reference for the underlying RAG infrastructure. |

## Stack

AWS Lambda · API Gateway · DynamoDB · Bedrock (Nova Pro) ·
Anthropic API · smolagents · FAISS

## License

TBD.
