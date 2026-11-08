# 12 Use case 1: tool-limit failure

**Goal:** prove that the pipeline stops a broken change from being approved, even when every code check passes.

**OWASP LLM risk:** LLM06 Excessive agency and LLM10 Unbounded consumption. A limit on tool rounds stops the assistant from looping forever.

In this use case you break the assistant on purpose. A **tool round** is one turn where the model asks for a tool, such as the policy search, and reads the result. The limit is `MAX_TOOL_ROUNDS` in `lambda/app/assistant.py`, 5 in the working app. Nothing measured sets 5: it was chosen as enough rounds for a question that needs several lookups, and few enough to stop a loop quickly. You cut it to 1, for the reason in the next section.

The code still runs and the unit tests still pass, so the pull request looks fine. After you merge it, the test that asks the real app questions fails. The `approved` pointer stays on the last good version. Then you repair the change with a second pull request.

In this lab, the staging environment is yours alone, so breaking it on purpose harms nobody. A company would not break a staging environment the whole team shares. It would run an exercise like this in a temporary copy, created for the test and deleted after.

This use case fails the same way every time, so run it first. The three use cases each break a different layer: 12 the code, 13 the prompt, 14 the search. Each one maps to a different OWASP risk.

## How the tool loop works

`lambda/app/assistant.py` gives the model up to `MAX_TOOL_ROUNDS` turns. On the first turn, the code forces the model to call a tool, such as the policy search, so it never answers a policy question from memory. On the next turn, it writes the answer from what the tool returned.

With only one round, the policy search uses up that one turn. No turn is left to write the answer. That is why the break uses 1: it fails every time, not only sometimes. So the employee gets this message: `Sorry, I couldn't finish that answer. Please try again.`

## Branches in this use case

- `test/tool-limit` sets `MAX_TOOL_ROUNDS = 1`. After you merge it, the live test fails and the version is not approved.
- `fix/tool-limit` sets `MAX_TOOL_ROUNDS = 5`. After you merge it, the live test passes and the version is approved.

Both branches start from the latest `main`. Each one goes back into `main` through its own pull request.

## Step 1: Start from a clean, passing main

```bash
cd ~/hr-rag-lab
git switch main
git pull --ff-only
git status --short
grep -n '^MAX_TOOL_ROUNDS' lambda/app/assistant.py
```

**Expected:** `git status --short` prints nothing, and the last line is `54:MAX_TOOL_ROUNDS = 5`. The line number is where the setting sits in the current code. If yours differs, use the number `grep` printed.

Before you break anything, write down three things from your last passing run:

- The commit: run `git log --oneline -1`.
- The prompt version: in Langfuse, open **Prompts**, then `hr-policy-system`. Note the version labelled with your last release, such as `v1.0.0`.
- The Lambda version that the `approved` alias points to: the command below prints it.

An **alias** is a name that points to one version of the Lambda function. This command prints the version behind `approved`:

```bash
aws lambda get-alias \
  --function-name "$(terraform -chdir=infra/terraform output -raw candidate_function)" \
  --name approved \
  --query FunctionVersion --output text
```

## Step 2: Break it

Create the branch and find the line:

```bash
git switch -c test/tool-limit
grep -n '^MAX_TOOL_ROUNDS' lambda/app/assistant.py
```

**Expected:** `54:MAX_TOOL_ROUNDS = 5`. The number before the colon is the line number.

In VS Code, open `lambda/app/assistant.py`, go to the line `grep` printed, and change it to:

```python
MAX_TOOL_ROUNDS = 1
```

This gives the assistant one round only. The policy search uses it up, so no turn is left to write the answer. The code still runs and the unit tests still pass, so only the pipeline's live questions can catch it.

Then open `VERSION`, at the top of the repository, and raise the last number by one, such as `1.0.0` to `1.0.1`. Every change that reaches `main` needs a new release number. Without it, the `checks` job fails. You raise the last number because this change adds no feature: in semantic versioning (document 07), that is a patch. Save both files.

Check the change before you commit:

```bash
git diff
```

**Expected:** two changed lines: `MAX_TOOL_ROUNDS` from `5` to `1`, and `VERSION` from `1.0.0` to `1.0.1`.

```bash
git add lambda/app/assistant.py VERSION
git commit -m "Exercise: limit the agent to one tool round"
git push -u origin test/tool-limit
```

## Step 3: Open the pull request and merge it

1. In GitHub, open a pull request from `test/tool-limit` into `main`.
2. Watch the `checks` job. **It passes**, because the code itself is still valid.
3. Get reviewer approval and merge.

You merge a broken change on purpose here. It is a planned test in staging, with made-up data.

## Step 4: Watch the gate block the release

The **release gate** is the point in the pipeline where a failed test stops a version from being approved.

In GitHub, open the **Actions** tab. Click the **new** run on `main`.

**Expected:**

- `Test the real authenticated application` fails.
- Promptfoo shows failed assertions.
- `Approve the exact tested staging version` is **skipped**.
- The report says `Release gate: FAIL.`.
- `Production handoff (dry run)` is **skipped**.
- `Tag the approved release` is **skipped**, so release `v1.0.1` gets no tag.

The pull request is already merged. What the gate blocked is the **release**: the broken version is never approved.

Check that the `approved` alias did not move:

```bash
aws lambda get-alias \
  --function-name "$(terraform -chdir=infra/terraform output -raw candidate_function)" \
  --name approved \
  --query FunctionVersion --output text
```

**Expected:** the same version number you wrote down in step 1.

## Step 5: Inspect the cause in Langfuse

Sign in to the chatbot as Amara and ask:

> How many PTO days do employees receive each year?

The staging app always runs the newest version, through the `candidate` alias. So it shows the broken answer, even though the release is blocked.

In Langfuse, open the trace for that question. You see **Agent round 1** with its policy search, and no **Agent round 2**. The limit stopped the assistant before it could write the answer. On the **HR answer** step, the metadata says `stopped: tool round limit reached (1)`. Its output is the "couldn't finish" message.

You can see the same thing in the model gateway, which records every call the app makes to the AI model:

1. Open the gateway admin page, as you did in document 03, step 3.
2. Click **Logs**.
3. Find this question. Compare its model calls with the same question from your passing run. The broken version makes fewer calls.

## Step 6: Repair it with a new branch

The broken change is already in `main`. So the repair needs its own branch and its own pull request.

```bash
git switch main
git pull --ff-only
git switch -c fix/tool-limit
grep -n '^MAX_TOOL_ROUNDS' lambda/app/assistant.py
```

In VS Code, open `lambda/app/assistant.py`, go to the line `grep` printed, and change it back to:

```python
MAX_TOOL_ROUNDS = 5
```

Then open `VERSION` and raise the last number by one again, to `1.0.2`. The repair is a new release, so it needs a new number. Save both files.

Then check it:

```bash
git diff
```

**Expected:** two changed lines: `MAX_TOOL_ROUNDS` from `1` back to `5`, and `VERSION` from `1.0.1` to `1.0.2`. Release `1.0.1` failed the gate, so it never gets a tag. The repair is a new release, `1.0.2`.

```bash
git add lambda/app/assistant.py VERSION
git commit -m "Restore the working tool budget"
git push -u origin fix/tool-limit
```

Open a pull request from `fix/tool-limit` into `main`, get it reviewed, and merge it.

## Step 7: Confirm the repair

In GitHub, open the **newest** run on `main`.

**Expected:** every job is green, the report says `Release gate: PASS.`, and the `approved` alias now points to the new version:

```bash
aws lambda get-alias \
  --function-name "$(terraform -chdir=infra/terraform output -raw candidate_function)" \
  --name approved \
  --query FunctionVersion --output text
```

Ask the same PTO question again. In Langfuse, the new trace shows **Agent round 1** with the policy search. Then it shows **Agent round 2** with the answer, the inline judge and the answer guardrail. The **inline judge** is a second model call that checks the answer against the policy text before the employee sees it.

If a different case fails, open its trace in Langfuse to see why.

## Step 8: Find the new release and clean up

The repair passed the release gate, so the `tag-release` job tagged it `v1.0.2`. The broken release, `v1.0.1`, has no tag.

```bash
git switch main
git pull --ff-only
git tag --list 'v*'
git branch -D test/tool-limit fix/tool-limit
```

**Expected:** `git tag` prints `v1.0.0` and `v1.0.2`.

## Evidence to keep

- The bad commit and the fix commit.
- The link to the failing run and the link to the passing run.
- The two Langfuse trace links for the PTO question, before and after.
- The `approved` version before, during and after.

**Next:** [13 Use case 2: wrong prompt instruction](13-use-case-2-wrong-prompt-instruction.md)
