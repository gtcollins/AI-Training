# 02 Infrastructure setup

**Goal:** connect Langfuse, choose your names, and create the whole application in AWS with one setup command.

The setup script does everything in order. First it checks that your computer has the right tools. Then it packages the code and creates the AWS resources with Terraform. Next it loads made-up employees, prepares the policy documents for search, and publishes the web page. It creates the sign-in accounts. Finally it signs in and asks the app one real question, to prove everything works.

## Step 1: Create a Langfuse project and keys

Langfuse is the tool that records every request: the question, the tools used, each question to the AI model, and the judge's result. Create your Langfuse project first, because the next steps need its address and keys.

1. Open [Langfuse Cloud](https://cloud.langfuse.com) and sign in with GitHub or Google.
2. Create an organization, then a project named for the HR application.
3. Write down your region's API address. For a US project it is `https://us.cloud.langfuse.com`.
4. In the project, open **Settings → API keys** and create a key pair. The public key starts with `pk-lf-`. The secret key starts with `sk-lf-`.

## Step 2: Create your two local settings files

These commands copy two example files. You fill them in next. Git ignores both files, so they never reach GitHub.

```bash
cd ~/hr-rag-lab
cp infra/terraform/terraform.tfvars.example infra/terraform/terraform.tfvars
cp .env.langfuse.example .env.langfuse
```

## Step 3: Add your Langfuse keys to .env.langfuse

Open `.env.langfuse` in your editor. Fill in the two keys and the address from step 1:

```ini
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_BASE_URL=https://us.cloud.langfuse.com
```

These keys are private to you. In document 10 you will add the same two keys to GitHub.

## Step 4: Set your names in terraform.tfvars

Open `infra/terraform/terraform.tfvars` in your editor. Set these values, and replace `yourname` with something unique to you:

```hcl
aws_region = "us-east-1"
project_name = "hr-ci-yourname"
cognito_domain_prefix = "northwind-hr-yourname"
domain_name = ""
aurora_instance_count = 1
langfuse_base_url = "https://us.cloud.langfuse.com"
trace_synthetic_content = true
```

- `aws_region`: where everything runs. `us-east-1` is chosen because Amazon Bedrock offers the lab's AI models there: Amazon Nova Pro, Nova Lite and Titan Embeddings V2. The model gateway also sends some Nova Lite calls to `us-west-2`, set as `gateway_secondary_region` in `infra/terraform/variables.tf`.
- `project_name`: the start of every AWS resource name. If two students choose the same name, their resources clash.
- `cognito_domain_prefix`: part of the sign-in page address. It must be unique across all of AWS.
- `langfuse_base_url`: the same Langfuse address as in `.env.langfuse`. The setup stops if the two are different.
- `domain_name`: leave it empty. The app then uses its CloudFront address, which starts with `https://` and ends in `cloudfront.net`. A company would put its own web address here.
- `aurora_instance_count`: the number of database servers. `1` costs half as much as `2`. With `2`, a second server takes over if the first one fails. The lab uses `1` to save money. A company running real HR data would use `2`.
- `trace_synthetic_content`: `true` lets Langfuse store the made-up questions and answers, so you can read them.

Leave the other lines in the file as they are. `alarm_email` stays empty: document 15 connects your email to the alarms another way.

## What the setup builds

One command creates the whole application in the AWS account you use for this lab. These are the main parts:

- **Terraform:** the tool that creates the AWS resources from the files in `infra/terraform/`. It keeps a record of what it created, called a state file, so it can change or delete those resources later.
- **Cognito:** the sign-in service. It holds the demo employees and the pipeline's test accounts.
- **API Gateway and Lambda:** API Gateway receives each question and checks the person's sign-in. Then a Lambda function, which runs the Python code, writes the answer.
- **Aurora PostgreSQL:** the employee database. Row-level security makes sure each person reads only what their role allows.
- **OpenSearch:** the search engine for the policy documents. **Indexing** means splitting each document into short passages and storing them, so the app can search them.
- **Amazon Bedrock:** the AWS service that provides the AI models and the guardrail safety filter.
- **LiteLLM gateway:** a small server that every question to an AI model passes through.
- **S3 and CloudFront:** S3 stores the web page files. CloudFront serves them to browsers over HTTPS.
- **Secrets Manager:** stores passwords and keys, so they are never written in the code.

## Step 5: Start the setup

This step creates AWS resources that cost money while they run.

```bash
bash scripts/setup-lab.sh
```

First the script checks your computer and your AWS account (`scripts/check-setup.py`):

- **Tools:** Python 3.10 or later, AWS CLI version 2, Terraform 1.9 or later, and Node.js with `npm` from document 00. These are the lowest versions the script accepts. Terraform's comes from `required_version` in `infra/terraform/versions.tf`.
- **Settings:** both files from steps 3 and 4, and your Langfuse keys.
- **AWS:** you are signed in to AWS in this terminal, the AI models are enabled for your account in Amazon Bedrock, and your `cognito_domain_prefix` is still free.

If anything is missing, it lists every problem at once. It stops before it installs or creates anything. Fix each problem and run the same command again.

Then each step prints one line with a green check mark and how long it took, such as `✓ Python environment (.venv) (2s)`. The full output of every step goes to the file `build/setup.log`.

## Step 6: Review the plan and answer y

Before Terraform changes anything, the script prints a plan summary and asks a question. The summary looks like `Plan: 129 to add, 0 to change, 0 to destroy.` and the question is `Apply these changes? [y/N]`. Your number of resources may be slightly different. Before you answer, check three things:

- On a first run, the summary says `0 to change, 0 to destroy`. Anything else means you are pointing at the wrong account or the wrong saved state.
- If the script lists resources under **These will be deleted or recreated**, read that list. On a first run it must be empty.
- The resource names in `build/setup.log` start with your own `project_name`.

If all three are true, type `y` and press Enter. Any other answer stops the script, and nothing changes.

## Step 7: Let the setup finish

Creating the AWS resources takes the longest. A timer shows how long it has been running. Then the script continues on its own:

1. Creates the database tables, the demo employees and the pipeline's test accounts.
2. Starts the LiteLLM model gateway and waits until it is ready.
3. Uploads the policy documents. Uploading `manifest.json` last starts the indexing. This file lists every document with its title, its version, whether it is current or replaced, and who may read it.
4. Builds the web page and uploads it to S3, behind CloudFront.
5. Creates the sign-in accounts and stores their passwords in Secrets Manager.
6. Stores your Langfuse keys in Secrets Manager.
7. Checks that indexing finished. Then it signs in as a test account and asks one question.
8. Deletes the chat from that check, so the app starts with an empty history.

**Expected:** `Setup complete.`, followed by the app address, the demo employees and their password.

If a step fails, the script prints a red ✗, what to check, and the most important error line. Fix the cause and run the same command again. Running it again is safe. Steps that have nothing new to do are skipped, and the others leave the working parts as they are.

## Step 8: Open the application

After `Setup complete.`, the script prints the app address, which starts with `https://`. It then lists the four demo employees you can sign in as. It shows each one's role and the one password they all share:

- `amara.diallo@northwind.example`: employee
- `liam.fischer@northwind.example`: manager
- `priya.raman@northwind.example`: hr
- `dana.whitfield@northwind.example`: exec

It also prints the sign-in for the LiteLLM gateway admin page, which you open in document 03.

The same details are saved in the file `demo-sign-in.txt` in the project folder. You can open it in your editor at any time. It holds the gateway's master key, so it stays on your computer. If you delete it, run `python scripts/show-demo-users.py` to create it again.

Setup also creates the file `build/github-settings.txt`. It collects every value GitHub will need in document 10, each under the exact name GitHub expects. Document 06 adds the last value. Like everything in the `build/` folder, it never reaches GitHub.

If you closed the terminal or lost the output, print the details again:

```bash
cd ~/hr-rag-lab
cat demo-sign-in.txt
```

Open the app address in your browser. You should see the Northwind HR sign-in page.

The lab's Python scripts run in a virtual environment: a folder, `.venv`, with the exact Python packages the lab needs. The setup created it. Turn it on in every new terminal before you run the scripts:

```bash
source .venv/Scripts/activate 2>/dev/null || source .venv/bin/activate
```

**Next:** [03 Sign in and read your first trace](03-sign-in-and-first-trace.md)
