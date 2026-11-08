# 09 Add the deployment and release jobs

**Goal:** add the rest of the pipeline to the workflow file. Step 1 adds the `candidate` job, which deploys each merge to staging and evaluates it. Step 2 adds the two release jobs that run after it passes. Document 07 explains how these jobs work together.

Here you only change a file on your computer. You commit it in document 10, after adding the GitHub settings the new jobs read. Without those settings, the first run would fail.

## Step 1: Append the candidate job

This command **adds** the `candidate` job to the end of the workflow file from document 04. The `>>` means "add to the end", so the lines already there stay as they are. Run it once only. Running it twice adds the jobs twice.

```bash
cd ~/hr-rag-lab
cat >> .github/workflows/hr-quality.yml <<'EOF'
  # Deploy the reviewed main commit, evaluate it, and approve only passing evidence.
  candidate:                                                            # job
    name: Deploy and evaluate staging                                   # job title
    # Secrets and AWS access are withheld from unreviewed pull-request code.
    if: >-                                                              # run condition
      always() && github.event_name != 'pull_request' && github.ref == 'refs/heads/main' &&
      needs.checks.result == 'success'
    needs: checks                                                       # depends on checks
    runs-on: ubuntu-24.04                                               # agent
    environment: hr-staging                                             # deploy environment
    timeout-minutes: 120                                                # time limit
    permissions:                                                        # token rights
      contents: read                                                    # read-only code
      id-token: write                                                   # AWS sign-in (OIDC)
      pull-requests: write                                              # PR comments
    env:                                                                # variables
      AWS_REGION: ${{ vars.AWS_REGION }}                                # setting
      HR_STAGING_FUNCTION: ${{ vars.HR_STAGING_FUNCTION }}
      HR_EVALUATION_FUNCTION: ${{ vars.HR_EVALUATION_FUNCTION }}
      HR_STAGING_ORIGIN: ${{ vars.HR_STAGING_ORIGIN }}
      HR_TEST_USERS_SECRET_ARN: ${{ vars.HR_TEST_USERS_SECRET_ARN }}
      HR_COGNITO_POOL_ID: ${{ vars.HR_COGNITO_POOL_ID }}
      HR_COGNITO_CLIENT_ID: ${{ vars.HR_COGNITO_CLIENT_ID }}
      LANGFUSE_PUBLIC_KEY: ${{ secrets.LANGFUSE_PUBLIC_KEY }}           # secret
      LANGFUSE_SECRET_KEY: ${{ secrets.LANGFUSE_SECRET_KEY }}
      LANGFUSE_BASE_URL: ${{ vars.LANGFUSE_BASE_URL }}
      HR_STAGING_FRONTEND_BUCKET: ${{ vars.HR_STAGING_FRONTEND_BUCKET }}
      HR_STAGING_DISTRIBUTION_ID: ${{ vars.HR_STAGING_DISTRIBUTION_ID }}
      VITE_AUTH_MODE: cognito                                           # build setting
      VITE_COGNITO_AUTHORITY: ${{ vars.VITE_COGNITO_AUTHORITY }}
      VITE_COGNITO_CLIENT_ID: ${{ vars.HR_COGNITO_CLIENT_ID }}
      VITE_COGNITO_DOMAIN: ${{ vars.VITE_COGNITO_DOMAIN }}
      EVAL_REPEATS: '2'                                                 # repeat each test
      EVAL_MAX_LATENCY_SECONDS: '20'                                    # slower answer fails
      EVAL_MIN_CONSISTENCY: '0.75'                                      # two answers must agree
      DO_NOT_TRACK: '1'                                                 # telemetry off
      DEEPEVAL_TELEMETRY_OPT_OUT: 'true'                                # telemetry off
    steps:                                                              # steps
      - uses: actions/checkout@v7                                       # get code
        with:                                                           # inputs
          persist-credentials: false                                    # drop Git token
      - uses: actions/setup-python@v7                                   # install Python
        with:
          python-version: '3.12'                                        # version
      - run: python -m pip install -r ci/requirements.txt               # install packages
      # Catch hidden spaces or line breaks in pasted settings before any step uses them.
      - name: Check the GitHub settings                                 # step title
        env:                                                            # step variables
          VARS_JSON: ${{ toJSON(vars) }}                                # every variable
          LANGFUSE_PUBLIC_KEY: ${{ secrets.LANGFUSE_PUBLIC_KEY }}       # secret
          LANGFUSE_SECRET_KEY: ${{ secrets.LANGFUSE_SECRET_KEY }}       # secret
        run: python ci/check_settings.py                                # names any bad setting
      # Exchange GitHub identity for short-lived credentials in the scoped AWS role.
      - uses: aws-actions/configure-aws-credentials@v6                  # AWS sign-in
        with:
          role-to-assume: ${{ vars.AWS_STAGING_ROLE_ARN }}              # IAM role
          aws-region: ${{ vars.AWS_REGION }}                            # region
      - uses: actions/setup-node@v7                                     # install Node.js
        with:
          node-version: '22'
      # Build the UI from this commit and upload it to the staging frontend.
      - name: Build and deploy React to staging                         # step title
        run: |                                                          # script below
          npm --prefix frontend ci
          npm --prefix frontend run build
          python ci/deploy_frontend.py
      # Publish the prompt version and package it with the matching backend code.
      - name: Publish exact candidate prompt and package it
        run: python ci/release.py --publish                             # publish prompt, build zip
      - name: Deploy the new backend version to staging
        run: python ci/deploy_candidate.py                              # deploy Lambda version
      # Obtain fresh test-user tokens and run every reference case twice through the API.
      - name: Test the real authenticated application
        id: live                                                        # step ID
        run: python ci/evaluate_api.py                                  # API tests
      # Retrieval must return each case's expected policy, and the two answers must agree.
      - name: Check retrieval quality and answer consistency            # step title
        if: always() && steps.live.outcome != 'skipped'                 # run condition
        run: python ci/quality_checks.py                                # retrieval + consistency
      # Store captured answers and scores for version comparisons; no extra generation here.
      - name: Record the dataset experiment in Langfuse
        if: always() && steps.live.outcome != 'skipped'
        uses: langfuse/experiment-action@c7c4efe432e37d4e3693a018de469233c0018de4 # v1.0.12
        with:
          langfuse_public_key: ${{ secrets.LANGFUSE_PUBLIC_KEY }}
          langfuse_secret_key: ${{ secrets.LANGFUSE_SECRET_KEY }}
          langfuse_base_url: ${{ vars.LANGFUSE_BASE_URL }}
          experiment_path: ci/langfuse_experiment.py                    # script
          should_skip_sdk_installation: 'true'                          # skip install
          should_fail_on_regression: 'true'                             # fail on regression
          should_fail_on_script_error: 'true'                           # fail on error
          should_comment_on_pr: 'false'                                 # no PR comment
      - uses: actions/setup-node@v7                                     # install Node.js
        with:
          node-version: '22'
      - name: Independently assert captured API answers with Promptfoo
        if: always() && steps.live.outcome != 'skipped'
        run: python ci/prepare_promptfoo.py                             # build config
      # Assert expected facts, forbidden disclosures and sources in the captured answers.
      - name: Promptfoo evaluation action
        if: always() && steps.live.outcome != 'skipped'
        uses: promptfoo/promptfoo-action@04839e664f52212b877170219dab0d7ad2bf6440 # v1
        with:
          github-token: ${{ github.token }}                             # GitHub token
          config: ci/promptfoo.generated.json                           # config file
          promptfoo-version: '0.124.1'
          use-config-prompts: 'true'                                    # use config prompts
          force-run: 'true'                                             # always run
          fail-on-threshold: '100'                                      # 100% must pass
          no-share: 'true'                                              # no upload
          disable-comment: 'true'                                       # no PR comment
          no-cache: 'true'                                              # no cache
          no-table: 'true'                                              # short log
          no-progress-bar: 'true'                                       # short log
      # Model-graded checks on a sample: reference facts, faithfulness, relevancy, bias, toxicity.
      - name: Evaluate answer quality with DeepEval
        if: vars.ENABLE_DEEPEVAL == 'true'                              # optional
        run: |
          python -m pip install -r ci/requirements-demo.txt
          python ci/deepeval_check.py
      - name: Prepare Evidently numeric drift inputs
        if: always() && steps.live.outcome != 'skipped' && vars.ENABLE_EVIDENTLY == 'true'
        run: python ci/prepare_drift.py                                 # build CSV
      # Compare latency and answer-length distributions; drift is an investigation signal.
      - name: Evidently proxy drift report
        if: always() && steps.live.outcome != 'skipped' && vars.ENABLE_EVIDENTLY == 'true'
        uses: evidentlyai/evidently-report-action@7b255ad073b66f0a2f511393edf24d2907c15008 # v1
        with:
          config_path: ci/evidently_config.py:conf                      # checks
          input_path: build/drift-current.csv                           # this run
          reference_path: ci/data/drift-reference.csv                   # baseline
          output: build/evidently-report                                # output folder
          test_summary: 'false'                                         # no pass/fail
          save_dataset: 'false'                                         # skip data
          save_report: 'true'                                           # keep report
          upload_artifacts: 'true'                                      # attach report
          check-link: 'false'                                           # skip link check
      # Update the approved staging alias only after required preceding checks pass.
      - name: Approve the exact tested staging version
        if: success()                                                   # only if all passed
        run: python ci/promote_lab.py                                   # promote
      # Retain a sanitized report even when evaluation fails.
      - name: Publish public-safe report
        if: always()
        env:                                                            # step variables
          JOB_STATUS: ${{ job.status }}                                 # did any earlier step fail?
        run: |
          if test -f build/evaluation.json; then
            python ci/report.py
            cat build/report.md >> "$GITHUB_STEP_SUMMARY"
          else
            echo 'Evaluation did not complete; release is blocked.' >> "$GITHUB_STEP_SUMMARY"
          fi
      - name: Report on associated pull requests
        if: always()
        uses: actions/github-script@v9                                  # GitHub API script
        with:
          script: |                                                     # JavaScript below
            const fs = require('fs');
            if (!fs.existsSync('build/report.md')) return;
            const pulls = await github.rest.repos.listPullRequestsAssociatedWithCommit({owner:context.repo.owner,repo:context.repo.repo,commit_sha:context.sha});
            const body = 'HR staging evaluation report (review all workflow jobs before release)\n\n' + fs.readFileSync('build/report.md','utf8');
            for (const pr of pulls.data) await github.rest.issues.createComment({owner:context.repo.owner,repo:context.repo.repo,issue_number:pr.number,body});
      - uses: actions/upload-artifact@v7                                # save artifacts
        if: always()                                                    # always save
        with:
          name: hr-candidate-evidence                                   # artifact name
          # Raw answers stay out. Score files quote parts of answers; a company keeps them in restricted storage.
          path: |                                                       # files below
            build/release.json
            build/report.md
            build/function.zip
            build/previous-approved.json
            build/deepeval.json
            build/quality.json
          retention-days: 3                                             # keep 3 days
EOF
grep -nE '^  (checks|candidate):' .github/workflows/hr-quality.yml
```

**Expected:** the last command prints two jobs: `checks` on line 15 and `candidate` on line 41.

## How to read the new job

The steps match the list in document 07. A few parts of the YAML are new:

- `${{ vars.NAME }}` reads a GitHub **variable**, and `${{ secrets.NAME }}` reads a GitHub **secret**. You add both to the `hr-staging` environment in document 10.
- `EVAL_REPEATS`, `EVAL_MAX_LATENCY_SECONDS` and `EVAL_MIN_CONSISTENCY` set how many times each question is asked, the time limit per answer, and the consistency pass mark. Change them here to change the gate.
- `id: live` names the API test step, so later steps can ask how it ended.
- `if: always() && steps.live.outcome != 'skipped'` means: run this step even if an earlier step failed, as long as the API test ran. Without `always()`, GitHub skips every step after a failure, and you would see no results.
- **Check the GitHub settings** runs before AWS sign-in. `ci/check_settings.py` names any empty setting, or one with a hidden space or line break. `toJSON(vars)` passes all variables.
- `if: success()` on the approval step means: run only if every earlier step passed.
- `JOB_STATUS: ${{ job.status }}` on the report step tells `ci/report.py` whether any earlier step failed. If one did, the report says `Release gate: FAIL`, even when every answer passed. The report and the approval always agree.
- `if: vars.ENABLE_DEEPEVAL == 'true'` turns DeepEval on or off with a GitHub variable. Evidently has the same switch, `ENABLE_EVIDENTLY`.
- `actions/upload-artifact` keeps the listed files for 3 days, under **Artifacts** on the run page. The release jobs from step 2 read them there. Three days, set by `retention-days`, is a chosen value. It is long enough for those jobs and for downloads. It is short because the score files quote parts of answers, and anyone can download a public repository's files.

If a job name appears twice, you ran the block twice. Create the file again with document 04, step 2, then run this block once.

Step 2 adds two jobs that run only after the `candidate` job passes:

- **`production-handoff-dry-run`** checks the tested commit and its passing report, then prints what production would receive. It changes nothing.
- **`tag-release`** tags the tested commit in Git with its release number from the `VERSION` file, such as `v1.0.0`, and pushes the tag to GitHub. It is the only job allowed to write to the repository (`contents: write`), and it tags a commit only when the release gate passed.

## Step 2: Add the two release jobs

This command adds both jobs to the end of the workflow file. Run it once only.

```bash
cd ~/hr-rag-lab
cat >> .github/workflows/hr-quality.yml <<'EOF'
  # Demonstrate the approval boundary without deploying production infrastructure.
  production-handoff-dry-run:                                           # job
    name: Production handoff (dry run)                                  # job title
    if: github.event_name != 'pull_request' && github.ref == 'refs/heads/main'
    needs: candidate                                                    # depends on candidate
    runs-on: ubuntu-24.04                                               # agent
    environment: hr-production-approval                                 # approval gate
    permissions:                                                        # token rights
      contents: read                                                    # read-only code
    steps:                                                              # steps
      - uses: actions/download-artifact@v8                              # get artifacts
        with:
          name: hr-candidate-evidence                                   # artifact name
          path: evidence                                                # folder
      - name: Verify exact tested release and show production handoff
        env:                                                            # step variables
          EXPECTED_COMMIT: ${{ github.sha }}                            # tested commit
        run: |
          python - <<'PY'
          import json, os
          from pathlib import Path
          release=json.loads(Path('evidence/release.json').read_text())
          report=Path('evidence/report.md').read_text()
          if release['commit'] != os.environ['EXPECTED_COMMIT']:
              raise SystemExit('Release does not match the merged commit')
          if 'Release gate: PASS.' not in report:
              raise SystemExit('A passing evaluation report is required')
          print('DEMONSTRATION ONLY: production would receive the tested backend artifact and exact prompt version.')
          print('React would be built from this same commit with production configuration.')
          print('No production resources or traffic are changed by this dry-run job.')
          PY
  # Name the approved release in Git: tag the tested commit with its VERSION number.
  tag-release:                                                          # job
    name: Tag the approved release                                      # job title
    if: github.event_name != 'pull_request' && github.ref == 'refs/heads/main'  # run condition
    needs: candidate                                                    # depends on candidate
    runs-on: ubuntu-24.04                                               # agent
    permissions:                                                        # token rights
      contents: write                                                   # may push a tag
    steps:                                                              # steps
      - uses: actions/checkout@v7                                       # get code
        with:                                                           # inputs
          fetch-depth: 0                                                # all history and release tags
      - uses: actions/setup-python@v7                                   # install Python
        with:
          python-version: '3.12'
      - uses: actions/download-artifact@v8                              # get artifacts
        with:
          name: hr-candidate-evidence                                   # artifact name
          path: evidence                                                # folder
      - name: Tag the tested commit with its release number             # step title
        env:                                                            # step variables
          EXPECTED_COMMIT: ${{ github.sha }}                            # tested commit
        run: python ci/tag_release.py                                   # create and push the tag
EOF
grep -nE '^  (checks|candidate|production-handoff-dry-run|tag-release):' .github/workflows/hr-quality.yml
```

**Expected:** the last command prints four jobs: `checks` on line 15, `candidate` on line 41, `production-handoff-dry-run` on line 217 and `tag-release` on line 248.

## How to read the release jobs

- `needs: candidate`: both jobs wait for the `candidate` job. If it fails, GitHub skips them. They then run at the same time.
- `environment: hr-production-approval`: the handoff job uses its own environment. In document 10 you can make a person approve it before it runs.
- `actions/download-artifact`: both jobs read the files the `candidate` job saved under **Artifacts**: `release.json` and `report.md`.
- `EXPECTED_COMMIT: ${{ github.sha }}`: the commit this run was started for. Both jobs stop if the saved release came from another commit. They also stop unless the report says `Release gate: PASS.` in full.
- `fetch-depth: 0`: downloads the existing tags, so `ci/tag_release.py` can refuse a release number that is already tagged.

When the handoff job runs, its log ends with three lines. The first starts with `DEMONSTRATION ONLY: production would receive`. The last is `No production resources or traffic are changed by this dry-run job.`

Pushing a tag does not start the workflow again: the workflow runs on pushes to branches only.

If a job name appears twice, you ran a block twice. Create the file again with document 04, step 2, then run the step 1 block and this block once each.

**Next:** [10 Environments, merge and first run](10-environments-merge-and-first-run.md)
