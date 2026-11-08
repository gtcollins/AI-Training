# 11 Read the evaluation results

**Goal:** find each tool's result from your first pipeline run, and understand what a pass or a fail tells you.

Document 08 explains what each tool is and what it checks. This document shows where to find each result and how to read it.

## How the live API test decides

`ci/evaluate_api.py` first asks one warm-up question, which is not timed or scored. Then it asks every test question twice. For each answer it checks five things:

- The answer contains the facts it must contain.
- It does not contain the facts it must not show.
- It names the expected policy as its source.
- It arrived within the time limit.
- It came from the exact version being tested.

A case passes only if **both** of its answers pass. One failed answer stops the version from being approved. At the end, pass or fail, the script deletes the test accounts' chats.

## What a failure means for each tool

- **The live test** fails with `slow`: one answer took longer than 20 seconds. Open that question in Langfuse and look for the step that took the longest, such as the policy search.
- **Retrieval** fails: the search did not find the policy a case expects. Open that question in Langfuse, click its **Policy search** step, and compare the passages it found with the policy.
- **Consistency** fails: the same question, asked twice, got two answers about different things. Read both answers in the Langfuse experiment, then open each one step by step. If the report lists a case under **Check by hand**, its two answers mention different numbers: compare them yourself.
- **Promptfoo** fails: at least one answer broke a rule of its case. Its log shows which rule.
- **Langfuse** fails: at least one case got a `reference_checks` score of 0, because one of its answers failed. Compare this run with your `v1.0.0` run to see which cases changed.
- **DeepEval** fails: the log names the case and the check that failed. For faithfulness, find the sentence that contradicts the policy passages. For relevancy, check whether the answer wandered away from the question. The grading model can be wrong, so read the answer yourself before you decide.
- **Evidently** shows a change: answers got longer or slower than in an earlier run. Find out why. A change does not prove an answer is wrong.

## Where to find each result

- **The overall result, case counts, speed, retrieval, consistency and DeepEval scores**: open the run in the **Actions** tab, click **Summary**, and read the report.

The report, written by `ci/report.py`, shows totals only. Its lines, from the top:

- **Cases passing every repetition:** such as `52/52`. A case counts only if both of its answers passed.
- **Illustrative 95% Wilson interval:** a statistics range, such as `93.1%–100.0%`. It says what pass rate the app might have on similar questions, given how many cases you tested. The 95% is the usual level in statistics. With the current 52 cases the range is wide. Treat it as a reminder that a fixed list of questions is a small sample, not as a measured error rate.
- **Application requests recorded:** the number of answers. With the current test cases, 52 questions asked twice give 104.
- The speed, retrieval, consistency and DeepEval sections, then the last line, **Release gate: PASS** or **FAIL**.

To see which case failed, open the step's own log, such as the Promptfoo step. There, `case_id` counts from 0, so `case_id` 40 is case 41 in the test case list.
- **Which retrievals missed, and which answers disagreed**: in the run, click the `candidate` job, then the **Check retrieval quality and answer consistency** step.
- **Promptfoo results**: in the run, click the `candidate` job, then the **Promptfoo evaluation action** step.
- **DeepEval scores**: in the run, click the `candidate` job, then the **Evaluate answer quality with DeepEval** step. The step's log shows each case's scores. For every score and the judge's reasons, scroll to **Artifacts** on the run page, download `hr-candidate-evidence` and open `deepeval.json`. `quality.json` in the same download holds the retrieval and consistency results.
- **Evidently report**: on the run page, scroll to **Artifacts** and download `evidently-results`. Open the HTML file inside.
- **Langfuse**: follow the steps below.

## Walk through your run in Langfuse

Open your Langfuse project and look at these four pages, in this order.

### 1. The prompt version

Open **Prompts → hr-policy-system**. The newest version is the one your run published. It has the label `candidate`. Its config shows which commit it came from.

### 2. The dataset

Open **Datasets**, then the dataset whose name starts with `hr-regression-`. Each item is one test question. Open an item to see the question, the facts it must contain, the facts it must not show, and its source. This is the "golden set" from week 6: the fixed list of questions every run is scored against.

### 3. The experiment for your run

In that dataset, open the list of runs. Each pipeline run adds one, named after its commit and its GitHub run ID. The run ID is the long number in the address of the run's page on GitHub. Open yours:

- Every question shows its two answers and a `reference_checks` score: 1 if both answers passed, 0 if not.
- On a passing run, every score is 1.
- After a use case breaks something, open two runs side by side. You see exactly which questions dropped to 0.

### 4. The traces behind one answer

Open **Traces** and set the time range to when your run happened. Each test question appears as its own trace, from a test account. Open one. You see the same step-by-step view as in document 03. Each agent round shows its model call and tool. Then come the inline judge and the guardrail check on the answer. Use it to follow a failing question step by step.

## What the thresholds mean

- **Promptfoo at 100%** means every question in this fixed list passed, every time. It does not mean the chatbot answers every possible question correctly.
- **20 seconds per answer** fails a slow release before people hit the API's 30-second limit, where they would get an error. The inline judge has to fit its check inside that same time. On the passing staging run, the slowest answer took 4.9 seconds.
- **Retrieval hit rate of 100%** means the search found the expected policy for every question in this list. The mean reciprocal rank shows how near the top of the results it was: 1 means always first.
- **Consistency of 0.75** is a chosen value between two staging measurements. The same answer in other words scored about 0.87, and a different answer about 0.07. It applies to policy answers only. It checks that two answers are about the same thing, not that their facts match: two answers with different numbers can still score high. The fact checks and the number comparison cover that.
- **DeepEval at 0.7** applies to every check, and higher is better. It is a starting value for this lab. Before a real company uses it, an HR expert must review the grading rules.
- **Evidently** compares the overall pattern of lengths and times between runs. Its stored reference comes from a made-up failing run, so expect differences.

When a case fails, look at the answer first, then at the check itself.

## Check your understanding

Open the report from your passing run and answer these:

1. How many cases passed both of their answers, and how many questions did the pipeline send in total?
2. Pick case 2 from [the test cases](reference-test-cases.md). Which fact must the answer **not** contain, and why?
3. DeepEval uses a model to grade only a small sample of cases, 5 by default. What could it catch that the fact checks miss, and what could it get wrong?

**Next:** [12 Use case 1: tool-limit failure](12-use-case-1-tool-limit-failure.md)
