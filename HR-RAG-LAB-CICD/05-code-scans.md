# 05 Code scans

**Goal:** add a second workflow that checks the code itself: its style, risky patterns, and the packages it uses.

The `checks` job from document 04 tests what the code does. The scans in this document read the code without running it. Companies usually keep them in their own workflow, next to the delivery workflow. Both workflows then run at the same time on every push and pull request.

## What each tool looks for

- **Ruff** (Python) and **ESLint** (React): **lint** tools. They find likely bugs and untidy code, such as an unused import or a React hook missing a value it depends on.
- **Bandit** (Python): risky patterns, such as running a shell command, opening a web address or writing to a shared folder. It rates each finding high, medium or low.
- **pip-audit** (Python packages) and **npm audit** (React app packages): compare each package version with a public list of known vulnerabilities.

**Why these tools:** each one is free and open source. Together they cover three concerns: style and likely bugs, risky code, and vulnerable packages. Choosing them was a judgement call for this lab. Other tools, such as Semgrep or Snyk, do similar work.

The tool versions are fixed, so a new tool release cannot change the results without anyone noticing. Ruff 0.17.1, Bandit 1.9.4 and pip-audit 2.10.1 are pinned in `ci/requirements-scan.txt`. ESLint and its plugins are pinned by `frontend/package-lock.json`. ESLint stays on version 9, because its React plugin does not support version 10 yet.

## Why the scans only report here

In this lab, the scans never fail a run. `ci/code_scan.py` runs each tool, writes what it found on the run's summary page, and always ends green. This lab is about the **release gate**: the step, explained in document 07, that approves a new version only when the app answers every test question correctly. Code scans are a separate concern.

A company usually goes further. It blocks a merge on high findings, and the team reviews the rest.

## Step 1: Add the workflow

The scan script and the tool lists are already in the repository: `ci/code_scan.py`, `ci/requirements-scan.txt` and `frontend/eslint.config.js`. You only add the workflow that runs them.

```bash
cd ~/hr-rag-lab
git switch feature/pipeline
cat > .github/workflows/code-scan.yml <<'EOF'
# Lint and security scans of the code itself. Report only: findings never stop a run.
name: Code scan                                                         # workflow name
on:                                                                     # trigger
  pull_request:                                                         # on pull request
  push:                                                                 # on push
    branches: [main, 'feature/**', 'fix/**', 'test/**']                 # branch filter
  workflow_dispatch:                                                    # manual run button
permissions:                                                            # token rights
  contents: read                                                        # read-only code
jobs:                                                                   # three jobs, side by side
  lint:                                                                 # job
    name: Lint Python and React                                         # job title
    runs-on: ubuntu-24.04                                               # agent (pinned image)
    steps:                                                              # steps
      - uses: actions/checkout@v7                                       # get code
        with:                                                           # inputs
          persist-credentials: false                                    # drop Git token
      - uses: actions/setup-python@v7                                   # install Python
        with:
          python-version: '3.12'                                        # version
      - uses: actions/setup-node@v7                                     # install Node.js
        with:
          node-version: '22'
          cache: npm                                                    # package cache
          cache-dependency-path: frontend/package-lock.json             # cache key
      - run: python -m pip install -r ci/requirements-scan.txt          # pinned scan tools
      - run: npm --prefix frontend ci                                   # ESLint and its plugins
      - run: python ci/code_scan.py lint                                # Ruff and ESLint
  python-security:                                                      # job
    name: Scan Python code and packages                                 # job title
    runs-on: ubuntu-24.04                                               # agent (pinned image)
    steps:                                                              # steps
      - uses: actions/checkout@v7                                       # get code
        with:                                                           # inputs
          persist-credentials: false                                    # drop Git token
      - uses: actions/setup-python@v7                                   # install Python
        with:
          python-version: '3.12'                                        # version
      - run: python -m pip install -r ci/requirements-scan.txt          # pinned scan tools
      - run: python ci/code_scan.py python-security                     # Bandit and pip-audit
  frontend-security:                                                    # job
    name: Scan React app packages                                       # job title
    runs-on: ubuntu-24.04                                               # agent (pinned image)
    steps:                                                              # steps
      - uses: actions/checkout@v7                                       # get code
        with:                                                           # inputs
          persist-credentials: false                                    # drop Git token
      - uses: actions/setup-python@v7                                   # install Python
        with:
          python-version: '3.12'                                        # version
      - uses: actions/setup-node@v7                                     # install Node.js
        with:
          node-version: '22'
          cache: npm                                                    # package cache
          cache-dependency-path: frontend/package-lock.json             # cache key
      - run: python ci/code_scan.py frontend-security                   # npm audit
EOF
grep -nE '^  (lint|python-security|frontend-security):' .github/workflows/code-scan.yml
```

**Expected:** the last command prints three jobs: `lint` on line 11, `python-security` on line 29 and `frontend-security` on line 41.

What the file does:

- `on:` and `permissions:`: The same triggers as the delivery workflow, and a read-only token. The scans need no AWS or Langfuse keys.
- `jobs:`: Three jobs with no `needs:`, so GitHub runs them at the same time. The scans are split by concern, lint, Python security and React security, so a slow scan does not hold up the others. That split is a judgement call; one job running all five tools would also work.
- `pip install -r ci/requirements-scan.txt`: Installs Ruff, Bandit and pip-audit at fixed versions.
- `npm --prefix frontend ci`: Installs ESLint and its React plugins from the lockfile.
- `python ci/code_scan.py ...`: Runs the tools for that job and writes the summary.

## Step 2: Commit, push and read the results

```bash
git add .github/workflows/code-scan.yml
git commit -m "Add code scans"
git push
```

On GitHub, open the **Actions** tab and click the newest **Code scan** run. The run opens on its **Summary** page, which shows one section per tool. Click **Details** under a section to see each finding.

Your pull request now shows 8 checks: the `checks` job and the three scan jobs, each once for the push and once for the pull request.

**Expected:** all three jobs finish green. With the current code, measured when this guide was written, the summary shows:

- **Ruff:** 60 findings, most of them about the order of imports.
- **ESLint:** 2 errors and 1 warning, in `Chat.jsx` and `SignIn.jsx`. They are left in the code on purpose, so you have real findings to read.
- **Bandit:** 0 high, 5 medium, 47 low.
- **pip-audit:** 0 packages with known vulnerabilities, for both requirements files.
- **npm audit:** 0 critical, 0 high, 0 moderate, 0 low.

The counts change when the code changes, or when the tools' vulnerability lists are updated.

## Step 3: Read the findings like a reviewer

A finding is a question for a person, not always a bug. On the run's Summary page, click **Details** under **Bandit**. It lists the five medium findings:

- `ingest_handler.py` writes to `/tmp`. In a Lambda, `/tmp` is the only folder the code may write to, and no other user shares it. This one is safe.
- `trace_exporter.py`, `check-setup.py` and `prepare-oidc.py` open a web address. Each address is one the lab sets itself, such as your Langfuse or GitHub address. An employee never types it. These are safe too.

The low findings are mostly the scripts calling `git`, `terraform` and `aws`. A team records each reviewed finding, for example with a `# nosec` comment in the code. The next scan then shows only new findings.

You can run the same scans on your computer, with the virtual environment turned on:

```bash
python -m pip install -r ci/requirements-scan.txt
python ci/code_scan.py python-security
```

**Expected:** the same Bandit and pip-audit sections as on the Summary page, printed in the terminal. pip-audit is the slowest scan, because it installs each requirements file in a temporary environment first. On GitHub, the whole Python security job took 34 seconds in one measured run.

## Step 4: Turn on secret scanning

**Secret scanning** finds keys and passwords pushed by mistake. **Push protection** goes further: it refuses the push before the secret reaches GitHub. Both are built into GitHub.

1. In your repository, open **Settings**, then **Advanced Security**.
2. Under **Secret Protection**, click **Enable**, if it is not on already.
3. Under **Push protection**, click **Enable**.

**Expected:** both show as enabled. Your Langfuse keys live in `.env.langfuse` on your computer, which Git ignores, and in GitHub settings from document 10. Push protection is the safety net if one ever ends up in a commit.

**Next:** [06 GitHub-to-AWS access](06-github-to-aws-access.md)
