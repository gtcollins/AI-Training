# Reference: troubleshooting

When a run fails, open it in GitHub's **Actions** tab and click the step with the red cross. The step's name tells you which part to check.

## "Not authorized to perform sts:AssumeRoleWithWebIdentity"

AWS refused to let the pipeline sign in. No HR test ran, so this is a sign-in problem, not a failed test.

The pipeline signs in to AWS with **OIDC**: GitHub gives the job a signed token, and AWS swaps it for short-lived access. The role's **trust policy** says which tokens AWS accepts.

Check that these four things match exactly:

1. `AWS_STAGING_ROLE_ARN` in the `hr-staging` environment is the role you created in document 06.
2. The `candidate` job has `id-token: write` and `environment: hr-staging`. The workflow you pasted already has both.
3. The role's trust policy audience is `sts.amazonaws.com`.
4. The trust policy subject contains your exact repository and ends with `:environment:hr-staging`.

Check the subject first. A subject with names only, such as `repo:USERNAME/hr-rag-lab:environment:hr-staging`, does not match a token that also carries numeric IDs, such as `repo:USERNAME@OWNER-ID/hr-rag-lab@REPO-ID:environment:hr-staging`.

To fix it, run the document 06 script again. It reads your repository's IDs from GitHub and updates the trust policy to match.

```bash
cd ~/hr-rag-lab
source .venv/Scripts/activate 2>/dev/null || source .venv/bin/activate
python scripts/prepare-oidc.py
```

When it prints `Done.`, click **Re-run failed jobs** in GitHub.

The role should trust only your own repository, so use your own repository's IDs.

## The deployment jobs show as skipped

- **On a pull request**: this is expected. Code that has not been reviewed gets no AWS access, so `candidate` runs only after the merge to `main`.
- `production-handoff-dry-run` after a failed `candidate`: this is expected too. The release gate stopped the release.
- `candidate` on `main`: the `checks` job failed first. Fix that, and `candidate` runs.

## The React build fails with a missing Cognito setting

The build stops when `VITE_COGNITO_AUTHORITY`, `VITE_COGNITO_CLIENT_ID` or `VITE_COGNITO_DOMAIN` is empty. The workflow takes `VITE_COGNITO_CLIENT_ID` from the `HR_COGNITO_CLIENT_ID` variable.

Print the right values, then add the missing variable to the `hr-staging` environment in GitHub:

```bash
cd ~/hr-rag-lab
grep -E '^VITE_COGNITO_(AUTHORITY|DOMAIN)=' frontend/.env.production
terraform -chdir=infra/terraform output -raw cognito_client_id
```

## A job fails because a variable is empty

The variables belong in the **hr-staging environment**, not under the repository's variables. Their names must match exactly, including capital letters. Compare your list with document 10, step 3, or with `build/github-settings.txt`.

## The setup script stops

The script prints a red ✗ next to the step that failed. Below it, it says what to check and shows the most useful error line. The full output is in `build/setup.log`.

Fix the cause, then run the same command again. Steps that already finished are skipped, or repeated safely:

```bash
cd ~/hr-rag-lab
bash scripts/setup-lab.sh
```

If it stops at the first step, **Tools, configuration files and AWS account**, nothing has been created yet. The script lists every problem it found. The common ones:

- **Missing infra/terraform/terraform.tfvars** or **Missing .env.langfuse**: you skipped document 02, step 2.
- **Replace "yourname"**: you copied `terraform.tfvars` but did not put in your own names.
- **Langfuse rejected the keys**: the keys in `.env.langfuse` are wrong, or `LANGFUSE_BASE_URL` points to another region.
- **AWS credentials do not work**: run `aws configure` or `aws sso login`, then try again.
- **cognito_domain_prefix is already taken**: someone else already uses that prefix. Choose another one in `terraform.tfvars`.
- **A stack named ... already exists**: that `project_name` is already deployed in this AWS account, from another folder. Run setup from that folder, or choose another name.
- **Secret ... is still reserved**: an earlier teardown with the same `project_name` left it behind. The message prints the command that frees it.

If the readiness check fails after the app is deployed, wait a minute and run setup again. The first search after the documents are indexed can be slow.

## The app breaks right after you rerun setup

The app reaches the chat function through the `candidate` alias. An **alias** is a name that points to one fixed version of the function, with the settings it had when it was published.

Running setup again can publish a new version. Only the pipeline moves `candidate`, so it keeps pointing at the older version and its old settings. If one of those settings points at something setup has replaced, every question fails, and so does the readiness check.

Compare the version behind the alias with the newest version:

```bash
cd ~/hr-rag-lab
FN="$(terraform -chdir=infra/terraform output -raw candidate_function)"
aws lambda get-alias --function-name "$FN" --name candidate --query FunctionVersion --output text
aws lambda list-versions-by-function --function-name "$FN" --query 'Versions[-1].Version' --output text
```

If the two numbers differ, point `candidate` at the newest version. Then run setup again, so the readiness check passes:

```bash
aws lambda update-alias --function-name "$FN" --name candidate \
  --function-version "$(aws lambda list-versions-by-function --function-name "$FN" --query 'Versions[-1].Version' --output text)"
bash scripts/setup-lab.sh
```

After document 10, your next merge to `main` also fixes it, because every pipeline run publishes a version and moves `candidate`. Moving `candidate` by hand is only for repairing setup. The `approved` alias is moved by the pipeline alone, once every check has passed.

## A test case fails and you do not know why

1. Find the failing case in the **Promptfoo evaluation action** step's log. The report gives only totals. Look the question up in [the test cases](reference-test-cases.md). Promptfoo numbers cases from 0, so `case_id` 40 is case 41 there.
2. In Langfuse, find the trace for that question. Use the time of the run and the test account to find it.
3. Open **Policy search**: did the right passage come back, and which searches ran?
4. Open **Inline judge**: what was its outcome? The **Inline grounding assessment** step under it shows the verdict and the reason.
5. Open each **Bedrock model call**: what exactly did the model receive, and what did it reply?

The cause is usually in the answer, the search or the grader. The expected answers come from the policy, so they stay as they are.
