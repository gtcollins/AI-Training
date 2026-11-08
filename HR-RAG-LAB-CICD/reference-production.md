# Reference: from this lab to production

This lab runs the whole life of an AI application, but at classroom size: one AWS account, one staging environment, and a release that becomes `approved` all at once. That keeps every run predictable, so the whole class sees the same thing.

A company running the HR assistant for real employees would add the practices below. For each one, this page says what the lab does, what production does, and how you could add it to this lab.

## 1. Separate environments and accounts

**In the lab:** one staging environment in your own AWS account. The production job is a dry run: it checks the tested version and prints what production would receive (document 16).

**In production:** staging and production live in **separate AWS accounts**. A mistake or a leaked permission in staging then cannot touch production data. The pipeline deploys to staging first. It promotes the same tested version to production only after the release gate passes and, often, after a person approves.

**How to add it here:** create a second account, and run the same Terraform there with production names. Then give the pipeline a second OIDC role that only the `hr-production-approval` environment may use. GitHub environments can require a named reviewer before a job runs, so the approval becomes a real person's click.

## 2. Infrastructure changes through the pipeline

**In the lab:** you run Terraform by hand from your computer, and its state file, the record of what Terraform created, stays in your folder.

**In production:** the state lives in shared storage, so the whole team works from the same record. Terraform's S3 backend keeps the state in an S3 bucket and uses a lock file, so two people cannot change the infrastructure at the same time. Every pull request shows `terraform plan`, the exact list of changes, for review. The merge applies it. Nobody changes production infrastructure from a laptop.

**How to add it here:** move the state to an S3 backend. Then add a pipeline job that runs `terraform plan` on pull requests and `terraform apply` after the merge. It signs in to AWS the same way as in document 06.

## 3. Releasing in steps: canary releases

**In the lab:** when a release passes, `approved` moves to it all at once.

**In production:** many teams first give a new version to a small share of real users. This is a **canary** release. If the new version behaves well, the share grows step by step to everyone. If it does not, the share goes back to zero.

### Split by person, not by request

A chat is a series of requests: a question, then a follow-up that depends on the first answer. If the first answer comes from one version and the follow-up from another, the conversation can contradict itself, and nobody can tell which version caused a problem. So a chat app splits traffic by **person** or by **conversation**, never request by request. This is called **sticky assignment**:

1. **Decide once.** When a conversation starts, the app turns the employee's ID into a number from 0 to 99 with a fixed hash. If the number is below the canary share, such as 10, the conversation gets the new version. The same person always gets the same number, so they get the same version on every visit.
2. **Remember the decision.** The app stores the chosen version with the conversation. Every follow-up in that conversation uses it, even if the share changes in the middle.
3. **Record it everywhere.** Every trace carries the version, as this lab's traces already carry their release. The team can then compare the two groups: errors, answer time, 👎 rate and judge scores.

The decision usually comes from a **feature flag service**. On AWS, AppConfig feature flags can split by a percentage of a value you pass in, such as the user ID. The same ID always gets the same answer. Teams also use LaunchDarkly or Unleash for the same job.

### What the flag switches

- **A prompt or model change:** the same deployed code holds both prompts or both model names. The flag picks one per conversation, so no second deployment is needed.
- **A code change:** two versions of the code run side by side, and a small routing step sends each conversation to the one the flag chose.

### How the rollout is gated

A person or a pipeline job widens the share in steps, such as 1%, 10%, 50%, then 100%. Before each step, it compares the two groups against written limits. For example: "the new version's 👎 rate and error rate are no worse than the old one's". If a limit is broken, the share goes back to 0%. New conversations then all start on the old version. Conversations already on the new version either finish there, or move at their next question.

### Where a per-request split still fits

A Lambda alias can also split traffic between two versions by weight. AWS CodeDeploy can raise that weight in steps, and roll it back when a CloudWatch alarm goes off. That split is made for every request separately, so it fits calls that stand alone, such as a single document ingestion. It does not fit a conversation, for the reason above. CloudFront can also send a share of visitors to a test copy of a website, and keep each visitor there with a cookie. It allows at most 15% of traffic, and a session ends after at most an hour of inactivity. That suits website changes, not a chat that must stay on one version.

**How to add it here:** add a version column to the `conversations` table, which the chat function already writes when a conversation starts. Choose the version there with the hash rule above, and pass it to the answer code, which then picks the prompt or model. Add the version as a tag on every trace, so the Langfuse dashboard can show each group.

## 4. A rollback strategy

**In the lab:** `ci/rollback_lab.py` moves `approved` back to the previous version (document 16).

**In production:** a team decides in advance **what** can be rolled back, **who** may do it, and **when**:

- **What:** the code version, the prompt, the model choice in the gateway's settings, and the policy index. Each needs its own way back. A rolled-back code version that expects a newer prompt or index can break.
- **Who:** anyone on call may roll back without asking. Rolling forward again goes through the normal pipeline.
- **When:** a written rule, such as "roll back if the 👎 rate doubles or errors pass 1% for 10 minutes after a release". Rolling back first and investigating after is usually faster than fixing in place.

## 5. Evaluation in production

**In the lab:** the release gate checks a fixed set of test questions before each release. People's 👍 and 👎 reach Langfuse, and a person reviews the 👎 answers (document 15).

**In production:** three more practices are common:

- **Grade live answers automatically.** Langfuse can run an AI judge on a sample of real traces, such as 5% of them. It saves its scores next to the votes. It finds bad answers even when nobody votes.
- **Check the AI judge against people.** The HR expert's grades in the review queue are the human answers. Comparing them with the judge's scores shows how far the judge can be trusted. This is called **calibrating** the judge. Until it is done, the judge's pass mark is a guess.
- **Grow the test cases from real failures.** Every confirmed mistake becomes a new test case, so the release gate learns from production.

**How to add it here:** create an evaluator in Langfuse for the `HR answer` traces with a low sampling rate. Then export the review queue's `failure_cause` scores and compare them with DeepEval's results on the same answers.

## 6. Monitoring with targets

**In the lab:** CloudWatch alarms for errors, server errors, slow answers and every single 👎, and the Langfuse dashboard **HR assistant health** (document 03).

**In production:** teams write down **service level objectives** (SLOs), targets that say how good is good enough. For example: "95% of answers in under 5 seconds", "fewer than 0.5% errors", "at least 85% thumbs up". Alarms then fire on the **rate** of problems against the target, not on single events. A single 👎 does not wake anyone up. A rising 👎 rate after a release does.

**How to add it here:** a CloudWatch alarm can compare two metrics, such as 👎 votes divided by all votes. The lab already counts the 👎 votes, so a second metric filter for all votes is enough to alert on a share.

## 7. On-call, runbooks and postmortems

**In the lab:** the [runbook](reference-runbook.md) lists what to do when the gate fails, an alarm fires, or the gateway is down.

**In production:**

- **On-call:** one named person per week receives the alarms, through a paging tool instead of a shared inbox.
- **Severity levels:** for example, "the assistant is down for everyone" is urgent at any hour, while "one wrong answer" waits for working hours.
- **Runbooks:** every alarm links to its runbook entry. Runbooks are kept current, like code.
- **Postmortems:** after an incident, the team writes what happened, how they found it, and what will prevent it next time. A postmortem never blames a person. Its actions often become new test cases or new alarms.

## 8. Security beyond the lab

**In the lab:** sign-in, row-level security, the guardrail, the private network, OIDC instead of keys, push protection, and code scans that report (documents 03, 05 and 06).

**In production:**

- Code scans **block** a merge on high findings, and a person reviews the rest.
- Keys and passwords are **rotated** on a schedule. The gateway master key and the test-account passwords in this lab never change.
- A **red team** tries new attacks on the assistant, beyond the fixed attack test cases. Successful attacks become new test cases.
- Logs and traces follow a written **retention** rule: how long they are kept, and who may read them.

## 9. Cost control

**In the lab:** every AI model call goes through the LiteLLM gateway. It caps spending at 5 US dollars a day for the app and 0.50 per person, and it limits each person's requests and tokens per minute. It records the cost of every call, by user and by purpose: `hr-chat` for answers, `inline-judge` for the live check, and `ci-judge` for the pipeline's judge. It also answers repeated questions from its cache. The Langfuse dashboard shows the model cost per day.

**This is the production pattern.** Companies control AI cost the same way. One gateway sits in front of every model, with budgets, rate limits and cost records per user, team or feature.

**What production adds:** two small things.

- **An alert when spending jumps,** before the budget stops people. For example, an alarm when today's cost is twice the usual day.
- **A budget alarm for the rest of the bill.** The gateway only sees model calls. The database, the gateway server and the private network connections cost money every hour too. AWS Budgets can email the team when the whole account's spending passes a set amount.
