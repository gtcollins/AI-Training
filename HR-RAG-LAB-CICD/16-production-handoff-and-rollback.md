# 16 Production handoff and rollback

**Goal:** see where the lab's approval in staging stops, and practise moving the `approved` pointer back to an earlier version.

**Staging** is the company's test copy of the app, where every change is tested before approval. **Production** is the copy real employees would use. This lab has no production copy, only a practice step that shows what would happen.

## What the production job does

The `production-handoff-dry-run` job starts only after the `candidate` job succeeds. It downloads the files the `candidate` job saved, and checks two things:

1. `release.json` names the same commit that was merged.
2. The report contains `Release gate: PASS.`

Then it prints what production would receive. It changes nothing in AWS and sends no users anywhere. That is why it is called a **dry run**.

To see it, open your last passing run in GitHub's **Actions** tab. Click the **Production handoff (dry run)** job and read the three lines it printed.

## From staging to production

Each stage of a release, first in this lab, then in a real company:

- **Validate**. This lab: deploy the new version to staging and test it there. Production: run the same checks on the version you plan to release.
- **Approve**. This lab: the `hr-production-approval` environment in GitHub, which can ask a person to approve. Production: a person allowed to approve releases.
- **Deploy**. This lab: a dry run only. Production: the tested backend package, the React build and the exact prompt version, with a separate production role and settings.
- **Verify and recover**. This lab: keep the staging results and the `approved` pointer. Production: test production after the deploy, watch for errors and bad answers, and go back to the last approved release if needed.

To make the job real, replace its dry-run step with steps that deploy to production and check it. Production then gets the exact package that was tested, not a new build.

## Two Lambda aliases

An **alias** is a name that points to one version of the Lambda function. The pipeline uses two:

- `candidate`: the version being tested. `ci/deploy_candidate.py` moves it on every merge. The staging app and API use it.
- `approved`: the last version that passed every check. `ci/promote_lab.py` moves it only after every required check passes. Production would deploy this one.

When `promote_lab.py` moves `approved`, it also notes the release and the version it replaced, in the alias description. For example: `Lab approved release pointer; release v1.0.2; previous approved version: 7`. The rollback reads that note, and prints which release `approved` moved from and to.

## Rollback: restore the previous approved version

A **rollback** means going back to the last version that worked. `ci/rollback_lab.py` reads the recorded version from AWS and moves `approved` back to it. It refuses any version that does not belong to the staging function. It never deletes anything.

**What it changes here:** only the `approved` pointer. The staging app follows `candidate`, so it keeps showing the latest merged version. In production, people are served from the approved version, and this is the step that puts them back on the last good one in seconds. To put this lab's staging app back on an earlier version, revert the change instead, as the [runbook](reference-runbook.md) shows.

This step is optional. It needs at least two passing runs, so that `approved` has an earlier version to go back to. The fix run from use case 1 counts as one.

### Step 1: Roll back

```bash
cd ~/hr-rag-lab
python ci/rollback_lab.py
```

**Expected:** two lines. The first names the releases, such as `approved now points to v1.0.0; it pointed to v1.0.2.` The second starts with `Rolled back`, then the function name and the two version numbers, such as `approved moved from version 9 to version 7`.

If you run it again, it swaps back, because each rollback notes the version it left. The next passing run on `main` also moves `approved` forward again.

## Evidence to keep

- The branch, pull request and run links for each change.
- The links to the failing and passing traces.
- The passing report.
- Your own explanation of the test results, and of how a release is approved and rolled back.

**Next:** [17 Tear down](17-tear-down.md)
