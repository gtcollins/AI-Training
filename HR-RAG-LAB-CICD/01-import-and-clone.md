# 01 Import and clone

**Goal:** get a working copy of the company's repository on GitHub and on your computer.

In a company, the repository and the app belong to the team. For this lab, you copy the repository into your GitHub account. That lets you set up its pipeline, the way the team's platform engineer would.

A plain `git clone` of the original repository is not enough. You need your own GitHub copy, because the pipeline's settings, environments and secrets live on the GitHub copy.

## Step 1: Import the repository into your GitHub account

1. Open [GitHub Import](https://github.com/new/import).
2. Source URL: `https://github.com/kserge2001/hr-rag-cicd-student-starter.git`
3. Owner: **your own account**. Repository name: `hr-rag-lab`. Use exactly this name: every command in the guide works in the folder `~/hr-rag-lab`.
4. Visibility: **Public**. All the data is made up. Also, private repositories need a paid GitHub plan for the secrets you add later.
5. Click to begin the import. When it finishes, the new repository's address must start with `https://github.com/YOUR-USERNAME/`.

If the import fails, open the original repository's page instead. Click **Use this template → Create a new repository**, with the same owner and name.

Then, in your new repository, open **Settings → Actions → General**. Actions is the GitHub feature that runs the pipeline. Under **Actions permissions**, check that **Allow all actions and reusable workflows** is selected.

## Step 2: Clone your copy and create the feature branch

These commands download your copy to your computer and create a branch to work on:

```bash
cd ~
git clone https://github.com/YOUR-USERNAME/hr-rag-lab.git
cd ~/hr-rag-lab
git remote -v
git switch -c feature/pipeline
```

`git remote -v` shows where the code came from. `origin` is the name Git gives the GitHub repository you cloned, and `git push` sends your commits there. The `origin` lines must show **your** repository. You stay on the `feature/pipeline` branch until document 10.

## Step 3: Look around

These are the folders you will use. Print their contents and the chatbot's instructions:

```bash
ls frontend lambda/app infra/terraform ci
cat lambda/app/prompts/hr_system.txt
```

- `frontend/`: the chat web page, built with React.
- `lambda/app/`: the Python code that answers questions: the chat handler, the policy search, the tools and the inline judge.
- `lambda/app/prompts/hr_system.txt`: the system prompt, the instructions the AI model gets before every question. Langfuse gives it a new version number on every release.
- `infra/terraform/`: the files that describe every AWS resource.
- `ci/`: ready-made pipeline scripts that package the code, deploy it, test it and score the answers.

In this lab you connect the `ci/` scripts to the pipeline. You do not write them.

**Next:** [02 Infrastructure setup](02-infrastructure-setup.md)
