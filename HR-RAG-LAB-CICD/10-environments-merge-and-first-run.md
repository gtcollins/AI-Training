# 10 Environments, merge and first run

**Goal:** give the new jobs their settings, merge your pull request, and watch the pipeline run from start to finish for the first time.

After this document, every merge to `main` does three things. It puts the new version on staging. It tests it by asking real questions as signed-in test users. It approves the version only if every required check passes.

## What a GitHub environment is

A GitHub **environment** is a named target that a job deploys to. Each environment has its own settings and rules:

- **Variables:** plain settings, such as a function name or the AWS region. Anyone who can open the repository can read them.
- **Secrets:** private values, such as API keys. GitHub stores them encrypted and hides them in logs. Once saved, nobody can read them back.
- **Deployment rules:** which branches may deploy there, and whether a person must approve each deployment first.

A job picks its environment with the `environment:` line in the workflow. The `candidate` job uses `hr-staging`. AWS accepts sign-ins only from jobs in that environment. The production job uses `hr-production-approval`. There, a rule can make a person approve before the job runs.

## Step 1: Create the two GitHub environments

In your GitHub repository, open **Settings → Environments → New environment**. Create these two:

- `hr-staging`
- `hr-production-approval`

## Step 2: Open the settings file

Every value you need here is in one file. Setup wrote it in document 02, and document 06 added the last value. Run this to bring it up to date:

```bash
cd ~/hr-rag-lab
source .venv/Scripts/activate 2>/dev/null || source .venv/bin/activate
python scripts/github-settings.py
```

**Expected:** `GitHub settings: wrote build/github-settings.txt (15 variables, 2 secrets).` If a line says `still missing`, it also names the document that creates that value.

Open `build/github-settings.txt` in VS Code. Each name is on one line, and its value is on the next line. They are in the same order as steps 3 and 4. Copy each name and value exactly as shown.

## Step 3: Add the staging variables

In GitHub, open **Settings → Environments → hr-staging**. Under **Environment variables**, click **Add environment variable**. Do this once for each name listed under `ENVIRONMENT VARIABLES` in the file:

- `AWS_REGION`, `AWS_STAGING_ROLE_ARN`
- `HR_STAGING_FUNCTION`, `HR_EVALUATION_FUNCTION`, `HR_STAGING_ORIGIN`
- `HR_TEST_USERS_SECRET_ARN`, `HR_COGNITO_POOL_ID`, `HR_COGNITO_CLIENT_ID`
- `HR_STAGING_FRONTEND_BUCKET`, `HR_STAGING_DISTRIBUTION_ID`
- `VITE_COGNITO_AUTHORITY`, `VITE_COGNITO_DOMAIN`, `LANGFUSE_BASE_URL`
- `ENABLE_DEEPEVAL`, `ENABLE_EVIDENTLY`

Paste only the value, with nothing before or after it. A copied value can bring a hidden space or line break, and GitHub keeps it without showing it. If that happens, the first step of the `candidate` job, **Check the GitHub settings**, fails and names the setting to fix.

`ENABLE_DEEPEVAL` and `ENABLE_EVIDENTLY` are on and off switches. With the value `true`, the DeepEval and Evidently steps run.

The two `VITE_COGNITO` values are not secret. The web app needs them to show the sign-in page, and its build stops if one is missing.

## Step 4: Add the two secrets

On the same `hr-staging` page, under **Environment secrets**, click **Add environment secret**. Add each name listed under `ENVIRONMENT SECRETS` in the file: `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY`.

These are the only secrets stored in GitHub. The pipeline signs in to AWS with the role from document 06, so it needs no AWS keys.

## Step 5: Protect the environments

- `hr-staging`: under **Deployment branches**, allow only `main`. If your GitHub plan offers it, also add a required reviewer.
- `hr-production-approval`: if your GitHub plan offers it, add a required reviewer.

If "prevent self-review" is on, the person who started a run cannot approve it. Then pick someone else as the reviewer. Write down any setting your plan does not offer.

## Step 6: Push and merge

```bash
git add .github/workflows/hr-quality.yml
git commit -m "Connect staging deployment and AI evaluation gate"
git push
```

1. Open your pull request from document 04. The push starts the `checks` job again, and it must pass. A run for a pull request only runs `checks`: no deployment and no AI evaluation.
2. The deployment jobs show as **skipped** on the pull request. This is expected. Pull requests run without AWS access, and the deployment jobs start after the merge.
3. If someone reviews your work, ask them to approve. Otherwise review the changes yourself, then click **Merge pull request**.

## Step 7: Watch the first release run

1. Open the **Actions** tab. Click the **newest** run on `main`: the one your merge started. Only a run on `main` deploys and tests the release. The runs for your branch and your pull request only ran `checks`, so they are the wrong place to look.
2. If the run asks for approval to deploy to `hr-staging`, approve it.
3. Click the `candidate` job and watch its steps run in order. It publishes the prompt, deploys the new version, and asks every test question twice: 104 answers with the current test cases. Then it runs the retrieval and consistency check and the evaluation tools. Last, it approves the version and writes the report.
4. Click **Summary** on the run page to read the report.
5. If the run asks for approval for `hr-production-approval`, approve it. Then open the **Production handoff (dry run)** job. Its log ends with three lines, starting with `DEMONSTRATION ONLY: production would receive`.

**Expected:** all four jobs have a green tick: `checks`, `candidate`, the production handoff and `tag-release`. The report says `Release gate: PASS.`

After it passes, open the app in your browser and click **Sign in**. Check that the sign-in page opens. The pipeline tests do not check this page, so you check it by hand.

Next, check that both aliases now point to the version this run published. In the AWS console:

1. Check that the region at the top right is **N. Virginia (us-east-1)**.
2. Open **Lambda → Functions**. Click the chat function: its name is your `project_name` followed by `-chat`.
3. Open the **Aliases** tab.

**Expected:** two rows, `candidate` and `approved`, with the same version number. Click `approved` to see its description: `Lab approved release pointer; release v1.0.0; previous approved version: none`. On later runs, `none` becomes the version it replaced. The **Versions** tab lists every version the pipeline has published.

The same check from the terminal:

```bash
FN="$(terraform -chdir=infra/terraform output -raw candidate_function)"
aws lambda list-aliases --function-name "$FN" --query 'Aliases[].[Name,FunctionVersion]' --output table
```

**Expected:** a table with two rows, `approved` and `candidate`, and the same version number in both.

To see every question the pipeline sent to the AI model, open the LiteLLM page as in document 03, step 3, and click **Logs**. The pipeline's questions come from the test accounts, whose email addresses start with `qa.`.

If a step fails, click it to open its log and find the cause. For AWS sign-in errors, see [Troubleshooting](reference-troubleshooting.md).

## Step 8: Find release v1.0.0 everywhere

Your first passing run is release `v1.0.0`, the number in the `VERSION` file. The `tag-release` job tagged the tested commit `v1.0.0` and pushed the tag to GitHub. Bring the tag to your computer and look at it:

```bash
git switch main
git pull --ff-only
git log --oneline --decorate -3
git tag --list 'v*'
```

**Expected:** the first `git log` line shows `tag: v1.0.0`, and `git tag` prints `v1.0.0`.

The same release number shows in every other place:

- **GitHub:** your repository's **Tags** list shows `v1.0.0`.
- **Lambda:** in the AWS console, open your chat function's **Versions** tab. The newest version's description reads `v1.0.0 (commit ...)`.
- **Langfuse:** under **Prompts**, `hr-policy-system` version 1 has the label `v1.0.0`. Each trace from the run shows `v1.0.0` as its **Release**.

**Save for later:** the link to this passing run and the prompt version shown in Langfuse. The use cases compare their results with release `v1.0.0`, and put files back from it.

**Next:** [11 Read the evaluation results](11-read-the-evaluation-results.md)
