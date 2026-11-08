# 11. Resume statements and presenting a project

## LLM applications and RAG

- Designed and built a retrieval-augmented generation (RAG) HR policy assistant on AWS end to end, using Amazon Bedrock, OpenSearch vector search and Lambda to return cited, role-specific answers to employees.
- Built the document ingestion pipeline that chunks, embeds and indexes policy files from S3 into OpenSearch, with version metadata so answers come from the current policy.
- Implemented hybrid keyword and semantic retrieval, returning "not enough evidence" when the policy documents do not support an answer.
- Built and containerized a Slack-integrated AI assistant with Python, Slack Bolt and Docker.
- Implemented structured LLM outputs with JSON schemas, Pydantic validation and function calling, so downstream systems receive validated data.

---

## AI agents and Kubernetes

- Designed and implemented an AI-powered Kubernetes troubleshooting agent with LangChain, Claude and the Kubernetes MCP server, which reads pods, events, logs and deployments to diagnose image-pull errors, CrashLoopBackOff and failing probes, and recommends the fix.
- Integrated the agent with Slack through Slack Bolt, giving engineers an automated diagnosis in the channel where they ask, with the bot limited to the scopes it needs.
- Secured the agent with defense in depth: read-only MCP mode, a dedicated read-only ServiceAccount with Secrets denied by RBAC, and step and time limits, so it cannot change the cluster whoever is asking.
- Validated the agent against deliberately broken workloads (an unpullable image, a crash-looping container and mismatched probes), and it identified the root cause in each case.

---

## Model gateway, security and access

- Integrated LiteLLM as the model gateway, routing all model traffic to approved Bedrock models with per-team budgets, rate limits and automatic fallback.
- Enforced employee-level data access with Cognito authentication, role-based OpenSearch filters and Aurora PostgreSQL row-level security, before any data reached the model.
- Added input and output guardrails with Amazon Bedrock Guardrails, and tested the application against prompt-injection attempts.

---

## Evaluation and release quality

- Built automated evaluation gates in GitHub Actions with promptfoo, DeepEval and LLM-as-judge scoring, testing every release candidate against a 52-question golden set.
- Added an inline LLM judge that checks each answer against the source policy before it reaches the user.

---

## Observability and cost

- Deployed self-hosted Langfuse with Docker Compose and instrumented the Kubernetes agent to trace sessions, model calls, tool calls, token usage and cost per user.
- Moved system prompts into Langfuse prompt management, enabling versioned changes and rollback without redeployment, linked to user feedback scores.
- Built a telemetry pipeline that exports structured CloudWatch logs to Langfuse Cloud through a dedicated Lambda function.
- Designed observability for the self-hosted platform: per-request traces in a managed MLflow App inside the AWS account, with personal data masked by Presidio.

---

## Cloud infrastructure and CI/CD

- Provisioned the AWS environment with Terraform, over 100 resources including the VPC, Lambda, API Gateway, Cognito, OpenSearch, Aurora, S3 and CloudFront.
- Designed a serverless architecture with event-driven ingestion triggered by S3 uploads, eliminating idle compute cost.
- Built CI/CD pipelines in GitHub Actions that deploy React to S3 and Python to Lambda, authenticating to AWS with OIDC instead of long-lived keys.
- Implemented code scanning, secret scanning, push protection and pre-commit hooks, with protected staging and approval-gated production environments.
- Protected the application with a web application firewall, and limited operator access to the private model gateway to AWS Systems Manager Session Manager.

---

## Self-hosted inference (design)

- Designed a self-hosted LLM platform on Amazon EKS with Terraform and Argo CD, serving Qwen 27B on vLLM behind a LiteLLM gateway with per-team chargeback.
- Planned GPU node groups with Karpenter, KEDA autoscaling on queued requests, Prometheus and Grafana monitoring, and SHA-256 verified model releases in S3.

---

## Presenting a whole project

A project presentation takes about two minutes, in this order:

1. **The problem:** who had it, and why it mattered.
2. **The architecture:** the main pieces and how a request flows through them.
3. **Your part:** what you built or owned.
4. **The decisions:** one or two choices you made, and why.
5. **How you know it works:** the tests, checks or numbers.
6. **One problem you solved:** what broke, the cause and the fix.

**The whole project: the HR assistant**

Employees were waiting up to a day for answers to simple HR questions by email. So we built an assistant that answers them from the approved policies and from the employee's own record.

It's serverless on AWS. People sign in through Cognito, the React app is served from S3 and CloudFront, and each question goes through API Gateway to a Lambda function. The policies are indexed in OpenSearch, employee data lives in Aurora, and every model call goes through a LiteLLM gateway to Bedrock.

I went serverless because the traffic is uneven, so we don't pay for idle servers. For security, I enforced access in the data layer, with search filters and row-level security in Aurora, instead of trusting the prompt.

To know it works, every release has to pass 52 test questions through the live app before it can be promoted. A second model also checks each answer before the employee sees it.

One issue I hit: it told us there wasn't enough evidence on parental leave, even though the policy was there. The search was returning too few results, so the passage never reached the model. I fixed that and added the question to the test set.

**Owning a piece: the infrastructure of the HR assistant**

On the HR assistant, my part was the infrastructure and the deployment pipeline. Someone else handled the AI code.

I wrote the Terraform for the whole AWS environment, over a hundred resources: the VPC, Cognito, API Gateway, the Lambda functions, OpenSearch and Aurora. I chose serverless so we weren't paying for idle servers.

Then I built the pipeline in GitHub Actions. Every pull request gets code scanning and secret scanning. Deployments authenticate to AWS through OIDC, so there are no AWS keys stored in GitHub. A merge to main deploys a release candidate to staging and runs the test questions, and production only goes out after a reviewer approves it.

The part that took the most work was the environment setup: keeping the staging and production variables and secrets separate.

**Owning every piece: the Kubernetes assistant**

Engineers kept asking the same Kubernetes questions in Slack: why is my pod in CrashLoopBackOff, why can't it pull its image, why isn't it ready? Each one pulled someone away from their own work. So I built a troubleshooting assistant that answers those questions right there in Slack.

You ask in plain English. The Slack bot passes the question to an agent built with LangChain and Claude. The agent gets its tools from the Kubernetes MCP server, which reads pods, events, logs and deployments, and the agent explains the cause and the fix. The model never touches the cluster itself: the MCP server does, with its own credentials.

The most important decision was access. The assistant is read-only, no matter who is asking. It connects with its own read-only ServiceAccount instead of the user's credentials, and it has no access to Secrets. So even when a cluster admin asks, or someone tries to trick it, it cannot change anything, and every fix stays with a person. I also gave the Slack bot only the scopes it needs, to read mentions and post replies.

To know it works, I tested it on apps I knew were broken. One had an image that could not be pulled, one was in CrashLoopBackOff, and one had probes that did not match the app. It found the cause each time. Later I added Langfuse, so every question shows up as a trace with its tool calls, tokens and cost.

One problem I hit: when I added the Langfuse tracing, the assistant stopped working. The tracing setup and the MCP server were competing for the same port. I moved the MCP server to another port and it came back.
