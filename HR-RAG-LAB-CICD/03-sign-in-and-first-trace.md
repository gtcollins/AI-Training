# 03 Sign in and read your first trace

**Goal:** use the chatbot as an employee, then follow that one question through Langfuse and the model gateway.

The trace you read here shows a normal, working answer. In the use cases later, you compare broken answers with it.

## Step 1: Sign in and ask a question

The app address, the demo employees and their shared password are in the file `demo-sign-in.txt`, in the project folder. Open it in your editor, or print it:

```bash
cd ~/hr-rag-lab
cat demo-sign-in.txt
```

Open the app address in your browser. Sign in as `amara.diallo@northwind.example`. Amara is an employee, so she can see only her own record.

Type this question and send it:

> How many PTO days do employees receive each year?

PTO means paid time off: vacation days.

Then, in the same conversation, ask a follow-up question:

> How many of those can I carry over?

**Expected:** with the current policy documents, the first answer says 25 days a year, rising to 28 after five years of service. The second says you can carry over up to 5 days. Both name the **Time Off Policy** as their source. The wording changes from one try to the next, but the numbers should not.

Every answer has a 👍 and a 👎 button under it. Under the second answer, click 👎. Type a short reason, such as `I wanted my own balance`, then click **Send**. Your rating is saved next to that answer in Langfuse. A reviewer can then find every answer people did not like.

## How Langfuse records a request

Langfuse uses four words:

- **Trace:** the record of one question, from the moment it arrives to the answer.
- **Span:** one step inside a trace, such as a search or a question to the AI model, with how long it took.
- **Tokens:** the small pieces of text a model reads and writes. Model providers charge per token, so token counts show the cost of each call.
- **Session:** all the traces of one conversation.
- **Prompt version:** the number of the system prompt version that produced the answer. The system prompt is the set of instructions the AI model gets before every question.

One of those steps is the **inline judge**. Before the employee sees an answer, the app asks the AI model a second question: is this draft supported by the policy passages it was given? If not, the draft is rewritten once. If the rewrite also fails, the employee gets a safe reply instead. The judge runs on every answer.

## Step 2: Find the trace in Langfuse

Langfuse shows what the app did with each question.

In Langfuse, open your project and click **Sessions**. Open the newest session. You should see:

- A coded user ID instead of the employee's email, which never appears. You also see the tags `employee` and `person`.
- Both questions in the same session.
- The **prompt version number** used for the answer, and the **Release**, such as `v1.0.0`, once the pipeline has deployed one.

Open one trace. Its steps form a tree, in the order they ran:

- **HR answer**: the question, the answer and its sources. Its metadata is a summary: how it ended, how many rounds and tools it used, how many tokens it used, and what each guardrail decided.
  - **Agent round 1**: the AI model's first turn.
    - **Bedrock model call**: everything the app sent to the AI model, its reply, its settings, the tokens and the cost. Here the reply is a request to search the policies, with the search words the model chose.
    - **Authorized HR tool**: the search request from the model, and the text the tool sent back.
      - **Policy search**: which documents this person's role may read, the filters, which searches ran, and each passage with a score for how closely it matches.
        - **Titan embedding** and **OpenSearch vector query**: the search itself, each with how long it took. Titan is an Amazon model that turns the question into a list of numbers that captures its meaning. OpenSearch then finds the passages whose numbers are closest.
  - **Agent round 2**: the AI model writes the answer from the passages.
    - **Bedrock model call**: the draft answer.
    - **Inline judge**: how the check ended, such as `passed first time`. Under it, **Inline grounding assessment** shows the judge's decision and its reason.
    - **Bedrock Guardrail: answer**: whether the guardrail let the answer through, hid parts of it, or blocked it, and what kind of data it found.

Click any step to see what went in, what came out, and its metadata. Langfuse highlights a step that has a warning, such as a judge that rejected a draft.

In Langfuse, a **score** is a grade attached to a trace. It can be yes or no, a number, or a category, and a person, a script or an AI judge can add it. Your 👎 is one.

Now open the trace of the second answer. Under **Scores**, you see `user_feedback` with the value `false` and your reason. If a reason contains personal data, such as a phone number, it is hidden before it is saved. To review every answer people disliked, open **Scores** or **Traces** and filter on `user_feedback`. A team can also send those traces to an **annotation queue**, a list where an HR expert grades each answer.

### The health dashboard

Setup also created a dashboard in Langfuse. Open **Dashboards**, then **HR assistant health**. Its six charts show how the app behaves for real people, not for the pipeline's test accounts:

- **Questions per day** from real people.
- **Answer time p95 by release:** 95% of answers took less than this. After document 10, each release gets its own bar, so you can compare them.
- **AI model cost per day**, by model.
- **Blocked questions per day:** questions the guardrail stopped.
- **Share of thumbs up** and **Votes: thumbs up and down**, from the 👍 and 👎 buttons.

Most charts are nearly empty for now. They fill up as people use the app. The script that builds the dashboard is `scripts/create-langfuse-dashboard.py`. If your setup ran before it existed, create the dashboard once:

```bash
cd ~/hr-rag-lab
python scripts/create-langfuse-dashboard.py
```

**Expected:** `Created the dashboard "HR assistant health" with 6 charts.` If it already exists, the script says so and changes nothing.

## Step 3: See the same questions in the model gateway

Every question the app sends to an AI model goes through the LiteLLM gateway. The gateway has an admin page that lists each of those questions with its model and its cost. It also shows whether the reply came from the gateway's cache: a store of recent replies it can reuse for an identical question. Langfuse shows the whole request from the app's side. The gateway shows each separate question to the AI model.

The gateway accepts connections only from the app itself, not from the internet. So you open its admin page through a tunnel: a private connection that AWS Systems Manager (SSM) creates from your computer to the gateway's port 4000. 4000 is the port LiteLLM listens on, set in `infra/terraform/gateway-bootstrap.sh.tftpl`. Your AWS permissions decide who can open it.

First, install the **Session Manager plugin**, the small AWS program that runs the tunnel. You only do this once.

On Windows, these commands download the installer and start it. Windows asks for permission to install: click **Yes**, then follow the installer.

```bash
curl -o ~/SessionManagerPluginSetup.exe https://s3.amazonaws.com/session-manager-downloads/plugin/latest/windows/SessionManagerPluginSetup.exe
~/SessionManagerPluginSetup.exe
```

On a Mac, with Homebrew:

```bash
brew install --cask session-manager-plugin
```

Close the terminal, open a new one, and check the install:

```bash
session-manager-plugin
```

**Expected:** `The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.` Other installs, such as Linux, are in [the AWS guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html).

Then open the tunnel:

1. Open a **second** terminal. The tunnel keeps this terminal busy until you close it.
2. In that terminal, go to the project folder and turn on the virtual environment from document 02. Every new terminal needs this before you run the lab's Python scripts.

```bash
cd ~/hr-rag-lab
source .venv/Scripts/activate 2>/dev/null || source .venv/bin/activate
python scripts/open-gateway-ui.py
```

The script prints the address, `http://localhost:4000/ui`, and the sign-in details. Open the address in your browser. Sign in as `admin` with the password it printed. The same details are also in `demo-sign-in.txt`. If port 4000 is already used on your computer, add `--port 4100` to the command and use that port in the address.

The password is the gateway's master key. It can create keys and change every limit, so it stays out of screenshots, tickets and chat messages.

Click **Logs** and find your two questions from step 1. Each question shows more than one model call: one for the answer and one for the inline judge.

When you are done, go to the second terminal and press Ctrl+C. This closes the tunnel.

## Step 4: See where the accounts live

The setup created two kinds of sign-in account in Cognito. Each kind has its own password:

- **Demo employees:** Amara, Liam, Priya and Dana. People sign in as these to use the app. You just signed in as Amara.
- **Test accounts:** `qa.employee`, `qa.manager`, `qa.hr` and `qa.exec`. Only the pipeline and the automatic tests sign in as these.

In the AWS console, open **Cognito → User pools**. Click the pool named after your `project_name`, ending in `-users`. Its **Users** tab lists both kinds of account.

This is how signing in protected the data in step 1:

1. Your browser got a sign-in token from Cognito.
2. API Gateway checked that token before the Lambda function ran.
3. The Lambda function read the employee's email from the checked token, never from the question.
4. Every database lookup and every search showed only what that employee may see.

## Step 5: How the pipeline signs in

The pipeline cannot click on a sign-in page. A script, `ci/auth.py`, signs in for it:

1. It reads the test accounts' password from Secrets Manager. The test accounts have their own secret there.
2. It signs in through Cognito and keeps the sign-in tokens in memory only.
3. It sends questions through API Gateway, so the same data permissions apply as for a person.

The pipeline's AWS role can read only the test accounts' secret. It cannot read the demo employees' password.

After each run, the pipeline deletes the test accounts' chats. Your chats as a demo employee stay as they are. To delete the chat history of every account yourself, run `python scripts/clear-chat-history.py`.

You should now see your two questions in one Langfuse session, with each agent round, the policy search and the inline judge. Each question to the AI model also appears in the gateway's logs.

**Next:** [04 Branch and pull-request checks](04-branch-and-pr-checks.md)
