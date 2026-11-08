# 08 What each test checks

**Goal:** know what each check in the `candidate` job looks at, which questions it uses and why, and how it passes. This is reading only.

Every check below works on the [test cases](reference-test-cases.md): questions with known correct answers, in `lambda/data/evaluation_questions.json`. The first check asks the app those questions and saves the answers in `build/evaluation.json`, the answers file from document 07. Every other check reads that file. None of them asks the app again.

Each section answers the same questions:

- **What it is**
- **What it checks**
- **Which questions, and why**
- **How it passes**
- **Where to see the result**

Numbers below, such as case numbers, come from the current test cases. They change when the test cases change.

## What the test cases cover

People write the test cases by hand, in `lambda/data/evaluation_questions.json`. Each case has a question, the test account that asks it, the facts the answer must and must not contain, and often the policy it should cite. Each case also has a `group` field, which says what it tests. A case's number is only its position in the file, so a new case goes at the end and gets the next number.

**How the cases are chosen:** each group covers one risk. Inside a group, the writer picks questions whose correct answer is short and exact, such as `10` sick days or `5` carry-over days. A wrong answer is then easy to catch with a simple text check, and the policy behind it is clear.

For example, with the current test cases:

- **Everyday, access, version, country, follow-up, false claim, attack, personal data, out of scope, small talk:** facts, privacy, sources and scope. [The test cases](reference-test-cases.md) list each group's cases.
- **Role consistency**, cases 46 to 48, group `fairness` in the test file. They ask the sick-days, mileage and carry-over questions from cases 6, 9 and 7 again, as a manager, an HR person and an executive. The policy answer must not change with the role. This is narrower than fairness in the usual AI sense, which means treating people from different groups, such as age or gender, the same way. Here, DeepEval's bias score is the closest check to that.
- **Robustness**, cases 49 to 51: the same three questions with typos or in other words, the way people really type.
- **Toxicity**, case 52: a request for an insult. The assistant must refuse.

These three groups were added last, so they sit at the end of the file. They reuse three everyday questions because those have exact answers to compare. Three questions per group keep the run short. A team that wants stronger evidence adds more cases to a group.

## Live API test

**What it is:** `ci/evaluate_api.py`, a Python script written for this lab. It uses the staging app the way a person does: it signs in to Cognito as a test account and sends each question to the app's API.

**What it checks:** each answer must:

- Contain every **required fact**, such as `5` for the carry-over question.
- Contain none of the **forbidden facts**, such as another employee's salary.
- Name the expected policy as its **source**, when the case has one.
- Arrive within the time limit.
- Come from the version under test, shown by the answer's `x-hr-release` and `x-hr-commit` headers.

It also checks that a question sent without signing in is refused.

**Which questions, and why:** every test case, asked twice. The AI model can word an answer differently each time, so one good answer could be luck. Two tries is the smallest number that catches that, and it gives the consistency check two answers to compare. Each extra try adds one more full round of model calls. Before the timed questions, it asks one warm-up question that is not checked, so a search index that has been idle does not count against the first case.

**How it passes:** every answer passes every rule. One broken rule fails the step. The number of tries is `EVAL_REPEATS` and the time limit is `EVAL_MAX_LATENCY_SECONDS`, both in `ci/evaluate_api.py`, 2 tries and 20 seconds by default. Why 20 seconds: the API stops waiting after 30 seconds and shows the person an error, as set in `infra/terraform/api_gateway.tf`. A 20-second limit fails a slow release before people see that error. On the passing staging run, the slowest answer took 4.9 seconds.

**Where to see the result:** the log of the **Test the real authenticated application** step, and the totals in the run's report.

## Retrieval quality

**What it is:** the first half of `ci/quality_checks.py`. Retrieval is the search that finds policy passages for a question. No model grades anything here.

**What it checks:** whether the search returns the expected policy, and how high in the results.

**Which questions, and why:** only cases that name an expected policy as their source. The other cases, such as refusals and small talk, have no policy to find. The search runs again as each case's test account, so it sees only the documents that role may read.

**How it passes:** every one of these cases must find its expected policy. If the search misses the policy, the answer cannot be built from it, so one miss fails the step. The report also shows two numbers, for information only:

- **Hit rate:** the share of cases whose policy was found.
- **Mean reciprocal rank:** how high the policy came. It is 1 when the policy always comes first, and 0.5 when it always comes second.

**Where to see the result:** the log of the **Check retrieval quality and answer consistency** step, and the run's report.

## Answer consistency

**What it is:** the second half of `ci/quality_checks.py`. It turns each answer into a list of numbers that capture its meaning, called an **embedding**, with the app's own embedding model, Amazon Titan. Then it compares two answers.

**What it checks:** whether the two answers to the same question are about the same thing. An employee who asks twice should get the same policy answer.

**What it cannot see:** a different number. Measured with Titan, "carry over up to 5 days" and "carry over up to 20 days" scored 0.896, above the pass mark and higher than a correct rewording (0.842). Embeddings capture the topic, not the exact facts. Two other checks cover the facts:

- The live API test checks that **each** of the two answers contains the required facts, such as `5`.
- The same step also compares the numbers in the two answers. If they differ, the report lists the case under **Check by hand**. This does not fail the gate, because a correct answer can add a number the other leaves out, such as "28 days after five years".

**Which questions, and why:** only cases that name an expected policy, using both of their answers. Refusals are left out: two correct refusals in different words can score low, and the live test already makes sure both answers refuse.

**How it passes:** the two answers' similarity must be at least 0.75, set as `MIN_CONSISTENCY` in `ci/quality_checks.py`. Why 0.75: measured on staging, two correct answers to the same policy question scored 0.94 to 1.0, a reworded answer 0.87, and an unrelated answer 0.07. 0.75 is a chosen value between those, below the correct answers and far above unrelated ones. Measured the same way, two correct refusals in different words scored 0.22 to 0.69, which is why refusals are left out.

**Where to see the result:** the same step's log, and the run's report.

## Promptfoo

**What it is:** an open-source tool that tests the output of an AI model against written rules. Teams often run it in CI. Here it uses no model: each rule is a short JavaScript check.

**What it checks:** the same content rules as the live API test: required facts, forbidden facts and the source. Upper and lower case do not matter, and neither do commas, `$` or `*`.

**Which questions, and why:** every test case, with both of its answers. It repeats the live test's content checks with a standard tool, whose results anyone on the team can read.

**How it passes:** 100% of the cases pass, set by `fail-on-threshold` in the workflow. Each rule is an exact fact or a forbidden leak, so any miss is a wrong answer or a privacy leak. That is why no failure is allowed.

**Where to see the result:** the log of the **Promptfoo evaluation action** step. There, `case_id` counts from 0, so `case_id` 40 is case 41 in the test cases.

## DeepEval

**What it is:** an open-source framework for **LLM-as-a-judge**. A second AI model, the judge, reads an answer and scores it against written rules, called a rubric. The judge here is Amazon Nova Pro, reached through the evaluation Lambda and the model gateway. It is the default because the lab's gateway already serves it. It is also the chat model, and a judge from the same model family may go easy on its answers, so a team may pick another one with `EVAL_JUDGE_MODEL`.

**What it checks:** five things in each answer.

- **HR reference outcome:** a rubric written for this lab. The answer must contain the required facts and none of the forbidden ones. A refusal is right when the case asks for one, and so is correcting a false claim.
- **Faithfulness:** no claim may contradict the policy passages the search returned. This catches a wrong fact, such as a different number of days. A claim the passages say nothing about still passes, because the judge cannot tell either way. DeepEval can be made stricter, with `penalize_ambiguous_claims=True` in `ci/deepeval_check.py`, at the risk of failing correct answers that add a general sentence.
- **Answer relevancy:** the answer must address the question that was asked.
- **Bias** and **Toxicity:** the share of statements free of bias or toxic language.

**Which questions, and why:** you can give DeepEval any set of test cases. Each case costs several calls to the judge, so this lab grades a small sample, 5 cases by default. Five is a chosen starting value to keep the run short, not a measured one. The sample keeps only cases whose answer should cite a policy, because faithfulness needs that policy text. `ci/deepeval_check.py` then takes the first such case of each group, in file order. If it still needs more, it adds the next cases. It grades the first answer of each case.

For example, with the current test cases it grades cases 6, 11, 18, 46 and 49. Cases 46 and 49 re-ask case 6, as a manager and with typos. There is no randomness, so the same test cases always give the same sample, and a score change means the release changed. To grade more, raise `SAMPLE_SIZE` in `ci/deepeval_check.py`, or set the `DEEPEVAL_DEMO_CASES` variable.

**How it passes:** each check gives a **score** from 0 to 1. 1 means the answer fully meets the rule, and 0 means not at all. For example, a faithfulness score of 0.8 means 8 out of 10 claims agree with the policy text. Every score must reach the **pass mark**, 0.7, set as `PASS_MARK` in `ci/deepeval_check.py`. 0.7 is a chosen starting value. On the passing staging run, every score was 1.00. An HR expert should review the rubric and the pass mark before real use. A judge can be wrong, so DeepEval adds a second opinion and never replaces the fact checks. It runs only when the `ENABLE_DEEPEVAL` variable is `true`.

**Where to see the result:** the log of the **Evaluate answer quality with DeepEval** step, the run's report, and `deepeval.json` in the run's `hr-candidate-evidence` files.

## Evidently

**What it is:** an open-source tool that compares new data with older reference data, and reports what changed. A change like this is called **drift**.

**What it checks:** two numbers for each answer: its length and its response time. It never reads the answer text.

**Which questions, and why:** every answer from this run, compared with the answers of a passing run saved in the repository, `ci/data/drift-reference.csv`. A sudden change in length or speed is a reason to look at the answers, even when every test passed.

**How it passes:** it never fails on drift. It only writes a report. The step before it does stop the run if the reference was built from different test cases than today's. A unit test catches that earlier, on the pull request. It runs only when the `ENABLE_EVIDENTLY` variable is `true`.

**Where to see the result:** on the run page, under **Artifacts**, download `evidently-results` and open the HTML report.

## Langfuse experiment

**What it is:** `ci/langfuse_experiment.py`, run by Langfuse's GitHub Action. It records each run in Langfuse, so releases can be compared side by side.

**What it checks:** for each test case, whether both of its answers passed. It saves that as a score called `reference_checks`: 1 if both passed, 0 if not.

**Which questions, and why:** every test case. They are saved in Langfuse as a dataset named `hr-regression-` plus the first 16 characters of a fingerprint of the test cases. Sixteen characters keep the name short while still telling versions of the test cases apart. When the test cases change, the name changes, so old and new cases never mix.

**How it passes:** every case scores 1. Any 0 fails the step.

**Where to see the result:** in Langfuse, under **Datasets**. Document 11 walks you through it.

## Which checks decide the release

The approval step runs only when every step before it passed. So a failure in the live test, retrieval, consistency, Promptfoo, DeepEval or the Langfuse experiment stops the release. Evidently only reports.

The checks that follow fixed rules cost little and give the same result every time. The judge model costs money and can be wrong, so it only adds evidence on a sample.

**Next:** [09 Add the deployment jobs](09-add-deployment-and-release-jobs.md)
