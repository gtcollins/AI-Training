# 04 Branch and pull-request checks

**Goal:** create the first job of the delivery workflow and see it run on a pull request.

This job runs every time you push code or open a pull request. It does four things: checks the release number, runs the unit tests, builds the backend package, and builds the React website. It gets **no AWS or Langfuse keys**. Pull-request code has not been reviewed yet, so it must not be able to touch AWS.

## What a CI workflow is

**CI** stands for continuous integration. It means every change is checked automatically before it is merged into `main`.

In GitHub Actions, a **workflow** is a YAML file in the `.github/workflows/` folder. It has three parts:

- **Triggers** (`on:`): the events that start the workflow, such as a push or a pull request.
- **Jobs:** groups of work. Each job runs on a fresh computer that GitHub provides, called a **runner**. One job can wait for another to finish.
- **Steps:** the commands inside a job, run one after another. A step either runs a shell command (`run:`) or a ready-made **action** (`uses:`). An action is a reusable step someone else wrote, such as one that downloads your code.

If you know Jenkins or Azure DevOps: jobs are like stages, runners are like agents, and actions are like plugins or tasks. In step 2, the comment at the end of each line says what that line does.

The job also stamps the backend package with a **release identity**. This is a `release.json` file inside the package. It records the release number, the commit the code came from, and a fingerprint of the prompt. The fingerprint is a SHA-256 hash: a 64-character code computed from the prompt text, which changes if any character changes. Later, you can trace every test result back to the exact code and prompt that were tested.

## Step 1: Run the unit tests locally

First, run on your computer the same tests the job will run:

```bash
cd ~/hr-rag-lab
source .venv/Scripts/activate 2>/dev/null || source .venv/bin/activate
python -m unittest discover -s ci/tests -v
```

**Expected:** every test ends with `ok`, and the last line is `OK`. Near the end, the output says how many tests ran. With the current tests that is `Ran 64 tests`. The number grows when someone adds a test.

The tests are the files in `ci/tests/`, currently seven. They check the prompt's structure, the release package, the inline judge and the connection to the model gateway. They also check the tracing code, user feedback, the rollback script, the release number, and that the Evidently drift reference matches the test cases. They use fakes instead of AWS and the AI model, so they need no access and run in seconds. To see an example, open `ci/tests/test_release.py`.

## Step 2: Create the workflow file

This command creates the file `.github/workflows/hr-quality.yml` with the first version of the workflow.

```bash
mkdir -p .github/workflows
cat > .github/workflows/hr-quality.yml <<'EOF'
# Reviewed changes, staging evaluation and release approval.
name: HR application delivery and evaluation                            # pipeline name
on:                                                                     # trigger
  pull_request:                                                         # on pull request
  push:                                                                 # on push
    branches: [main, 'feature/**', 'fix/**', 'test/**']                 # branch filter
  workflow_dispatch:                                                    # manual run button
permissions:                                                            # token rights
  contents: read                                                        # read-only code
concurrency:                                                            # one run at a time
  group: hr-staging                                                     # queue name
  cancel-in-progress: false                                             # never cancel
jobs:                                                                   # stages
  # Validate proposed changes before giving code deployment credentials.
  checks:                                                               # job
    runs-on: ubuntu-24.04                                               # agent (pinned image)
    steps:                                                              # steps
      - uses: actions/checkout@v7                                       # get code
        with:                                                           # inputs
          persist-credentials: false                                    # drop Git token
          fetch-depth: 0                                                # all history and release tags
      - uses: actions/setup-python@v7                                   # install Python
        with:
          python-version: '3.12'                                        # version
      - name: Install delivery helper dependencies                      # step title
        run: python -m pip install -r ci/requirements.txt               # install packages
      - name: Release version is new                                    # step title
        run: python ci/check_version.py                                 # VERSION newer than last tag
      - name: Prompt structure and release contract tests
        run: python -m unittest discover -s ci/tests -v                 # unit tests
      - name: Build backend candidate with release identity
        run: python ci/release.py                                       # build package
      - uses: actions/setup-node@v7                                     # install Node.js
        with:
          node-version: '22'
          cache: npm                                                    # package cache
          cache-dependency-path: frontend/package-lock.json             # cache key
      - run: npm --prefix frontend ci                                   # install UI packages
      - run: npm --prefix frontend run build                            # build UI
EOF
```

What each part does:

- `on:`: The workflow starts on pull requests, on pushes to `main`, `feature/**`, `fix/**` and `test/**`, and when you click **Run workflow** in GitHub. Those branch patterns match the branch names this lab uses: `feature/pipeline` here, and `test/...` and `fix/...` branches in the use cases of document 11. A push to a branch with any other name starts nothing.
- `permissions: contents: read`: The job can read the code and nothing else.
- `concurrency: hr-staging`: Only one run at a time. Two deployments never overlap, because there is only one staging environment: the test copy of the app that setup created in document 02. `cancel-in-progress: false` makes a new run wait instead of stopping the one already running, so a deployment is never cut off halfway.
- `runs-on: ubuntu-24.04`: The job runs on a GitHub runner with Ubuntu Linux 24.04. The name `ubuntu-latest` would move to a newer Ubuntu whenever GitHub decides. Naming the exact version keeps the runner the same until you change it.
- `persist-credentials: false`: The Git token is removed from the runner after the code is downloaded.
- `actions/checkout`: Downloads your repository's code onto the runner.
- `@v7` after an action's name: the major version of that action. The lab uses the newest major versions, which run on Node.js 24, because GitHub is retiring actions that run on Node.js 20.
- `fetch-depth: 0`: Downloads the full history, including the release tags, so the next step can compare versions. A release tag, such as `v1.0.0`, marks the commit of a release. The pipeline creates them, starting in document 10.
- `actions/setup-python` and `pip install -r ci/requirements.txt`: Installs Python 3.12 and the packages the pipeline scripts need. Python 3.12 matches the Lambda runtime, `python_runtime` in `infra/terraform/lambda.tf`, so the code is tested on the same Python it runs on in AWS.
- `Release version is new`: The `VERSION` file at the top of the repository holds the release number, such as `1.0.0`. `ci/check_version.py` checks that it is newer than every release tag, so a number is never used twice. Before the first release there are no tags. Then it prints `Release version v1.0.0: the first release.`
- `unittest discover -s ci/tests`: Runs the same unit tests as step 1.
- `ci/release.py`: Checks the prompt and builds the backend package, `build/function.zip`. It adds a `release.json` that records the release number, the commit and the prompt.
- `actions/setup-node`, `npm ci` and `npm run build`: Installs Node.js 22 and the website's packages, exactly as listed in `frontend/package-lock.json`, then builds the React website. A build error fails the job. Node.js 22 is a long-term support release; any version the setup accepts, 20 or 22 and later, would build the site. `npm ci` installs the exact versions in the lockfile, so every run builds the same website.

## Step 3: Commit and push

```bash
git add .github/workflows/hr-quality.yml
git commit -m "Add HR branch and pull-request checks"
git push -u origin feature/pipeline
```

## Step 4: Open the pull request

Open your repository on GitHub. Right after a push, a yellow banner appears above the file list. It shows `feature/pipeline` and a **Compare & pull request** button. Click the button, then click **Create pull request**.

If the banner is gone, open the **Pull requests** tab and click **New pull request**. Set **base** to `main` and **compare** to `feature/pipeline`. Then click **Create pull request**.

Near the bottom of the pull request page, you see the `checks` job twice: `checks (push)` for your push, and `checks (pull_request)` for the pull request. Click **Details** next to one to watch its log.

**Expected:** both finish green, and the page shows **All checks have passed**.

Leave the pull request open for now. Document 07 explains the deployment and release jobs, and in document 09 you add them to it.

**Next:** [05 Code scans](05-code-scans.md)
