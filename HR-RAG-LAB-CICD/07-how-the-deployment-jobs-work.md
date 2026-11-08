# 07 How the deployment jobs work

**Goal:** understand what the deployment jobs do, before you add them to your workflow in document 09.

There is nothing to run in this document.

## The big picture

Every merge to `main` starts the same chain of jobs. In plain words:

1. **Check** the code: run the tests and build it. This is the `checks` job from document 04.
2. **Deploy** the new version to **staging**. Staging is the company's test copy of the app, separate from production. The engineers, the HR expert who reviews the answers, and the pipeline all use it before a version is approved.
3. **Test** the deployed app by asking it real questions, the way an employee would.
4. **Decide:** if every test passed, mark this version as **approved**. If any test failed, stop. A failing version is never approved.
5. **Report** what happened, pass or fail.
6. **Hand off** to production. In this lab, this last job only prints what production would receive.

Step 4 is called the **release gate**: the point where one failed test stops the release. Steps 2 to 5 run in the `candidate` job. Step 6 is the `production-handoff-dry-run` job.

## How a version is deployed: Lambda versions and aliases

The app's backend runs on AWS Lambda. Lambda keeps every release as a separate, numbered copy:

- **Version:** a numbered copy of the code and its settings, such as version 7 or 8. Once created, a version never changes.
- **Alias:** a name that points to one version. You can move it to point at another version. The app is always called through the alias name. So moving the alias switches everyone to the new version at once.

If you know Git: versions are like commits, and aliases are like branches.

The chat function has two aliases:

- **`candidate`** points to the newest version, the one being tested. The pipeline moves it on every merge. The staging app you sign in to uses it.
- **`approved`** points to the last version that passed every test. The pipeline moves it only when the release gate passes.

So when a merge breaks something, `candidate` points to the broken version, and `approved` still points to the last good one. The staging app shows the broken answers, because it follows `candidate`. Documents 12 to 14 use this to show a failure.

In this lab, nothing serves `approved` yet. It is the pointer a production deployment would use, so you can see the gate keep a broken version away from it. The staging app keeps showing the latest merged version until you merge a fix or revert the change.

Right now the chat function has only `candidate`. The first passing run, in document 10, creates `approved`.

## What the candidate job does, in order

Each item below is one step in the workflow file, with the script or action it runs.

**Before it starts**
- **Runs only on `main`, and only after `checks` passed.** A pull request never gets AWS access or secrets. So code that has not been reviewed cannot touch AWS.

**Sign in and deploy**
- **Sign in to AWS** (`configure-aws-credentials`): uses the role from document 06. AWS gives the job temporary credentials that expire within the hour.
- **Deploy the website** (`ci/deploy_frontend.py`): builds the React website from this commit and uploads it.
- **Publish the prompt** (`ci/release.py --publish`): saves the system prompt in Langfuse as a numbered version, linked to this commit, and packages the backend code.
- **Deploy the backend** (`ci/deploy_candidate.py`): uploads the code as a new Lambda version, and moves `candidate` to it.

**Test**
- **Ask the real app**, also called the **API test** (`ci/evaluate_api.py`): signs in as the test accounts and asks one warm-up question that is not timed. Then it asks every question in [the test cases](reference-test-cases.md) twice. A wrong answer, or an answer slower than the time limit, fails the step. Document 08 explains the two tries and the 20-second limit.
- **Check retrieval and consistency** (`ci/quality_checks.py`): checks that the search finds each case's expected policy, and that both answers to each question are about the same thing.

**Evaluate the answers**
- **Langfuse, Promptfoo, DeepEval and Evidently** each score the saved answers in their own way. The next section explains each one.

**Decide and report**
- **Approve** (`ci/promote_lab.py`): moves `approved` to this version, but only if every step above passed.
- **Report** (`ci/report.py`): writes a summary on the run page and on the pull request. It never includes employees' answers.

If the API test (**Ask the real app**) fails, the retrieval and consistency check, Langfuse, Promptfoo, Evidently and the report still run, so you can see what broke. **Approve** is skipped, and so is DeepEval.

DeepEval and Evidently run only when the GitHub variables `ENABLE_DEEPEVAL` and `ENABLE_EVIDENTLY` are `true`. You set them in document 10.

## The answers file: build/evaluation.json

The API test saves every answer it gets in one file, `build/evaluation.json`, on GitHub's machine. With the current test cases, that is 52 questions asked twice, so 104 answers. For each answer it records:

- Which test case it was, and whether this was the first or second try.
- The full answer and the policy sources it cited.
- How long it took.
- Whether it passed, and if not, why, such as `missing:5` or `slow:22.1s`.

Every later check reads this file instead of asking the app again: retrieval and consistency, Promptfoo, DeepEval, Langfuse, Evidently, the approval step and the report. So every tool judges the same answers, and the model is asked only once per answer.

The file holds full answers, including the test accounts' salary figures. So it is never uploaded with the run's files. It disappears when the job ends. Only the summary and the score files are kept.

## What is checked, and by which tool

The checks after the API test each look at the answers in a different way: retrieval quality, answer consistency, Promptfoo, DeepEval, Evidently and the Langfuse experiment. The next document, [08 What each test checks](08-what-each-test-checks.md), explains each one: what it is, what it checks, which questions it uses and why, and how it passes.

## One release number everywhere

The `VERSION` file holds the release number, such as `1.0.3`. Each change raises it: the last number for a fix, the middle number for a new feature, the first number for a change that breaks something. This is called **semantic versioning**. The pipeline stamps the number on everything it builds, so each place shows which release it belongs to:

- **The package:** `release.json` records the version, the commit and the prompt.
- **Langfuse prompt:** the prompt version gets the label `v1.0.3`, next to `candidate`.
- **Lambda:** the published version's description reads `v1.0.3 (commit a21bc86)`. AWS still numbers versions 1, 2, 3, and the description says which release each one is.
- **Traces:** every trace's **Release** field shows `v1.0.3`.
- **API answers:** each answer carries the header `x-hr-release: v1.0.3`.
- **Git:** after the release gate passes, the `tag-release` job tags the tested commit `v1.0.3` and pushes the tag to GitHub. A release that fails the gate gets no tag.

## What these checks still do not cover

These gaps remain:

- **Real users:** the test cases are a fixed list. Real people can ask things no case covers. Their thumbs up and down reach Langfuse as scores, but those scores only point to answers to review. They do not stop a release.
- **Full reference answers:** each case lists required facts, not a complete ideal answer. So nothing compares a whole answer with an ideal one.
- **Human review:** the grading model's scores have not yet been compared with an HR expert's grades on the same answers.
- **Wider attacks:** there are four attack cases, but no systematic attack testing, called red-teaming.
- **Load:** the tests ask one question at a time, never many people at once.

## The production handoff job

This job starts only after the `candidate` job succeeds. It runs in the `hr-production-approval` environment, so a person can be required to approve it first (document 10). It checks two things: the tested commit is the one you merged, and the report says the gate passed. Then it prints what production would receive. It changes nothing.

**Next:** [08 What each test checks](08-what-each-test-checks.md)
