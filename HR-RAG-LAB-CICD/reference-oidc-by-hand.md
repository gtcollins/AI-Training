# Reference: OIDC by hand

In document 06, `scripts/prepare-oidc.py` does everything on this page with one command. Use this page if you want to see each step, or to build the same thing yourself in the AWS console.

**OIDC** is how the pipeline signs in to AWS without a stored password. GitHub gives each job a signed token. AWS checks the token and gives the job short-lived access, through an IAM **role**.

You end up with the same role, with the same name, either way. So documents 10 and 17 work the same.

## Step 1: Write the two policy files

With `--dry-run`, the script writes both files and checks them. It changes nothing in AWS:

```bash
cd ~/hr-rag-lab
source .venv/Scripts/activate 2>/dev/null || source .venv/bin/activate
python scripts/prepare-oidc.py --dry-run
```

It prints `Creating the role` followed by the role's name, such as `hr-ci-yourname-github-ci`, and ends with `Dry run: nothing was changed in AWS.` The name is your `project_name`, from `infra/terraform/terraform.tfvars`, followed by `-github-ci`. Write the name down for step 3.

The two files are in `build/`:

- `github-oidc-trust.json`: who may use the role. This is the **trust policy**. It allows only jobs in your repository's `hr-staging` environment, with the audience `sts.amazonaws.com`.
- `github-oidc-permissions.json`: what the role may do. It currently has six permissions, one for each service listed at the top of document 06.

## Step 2: Register GitHub as an identity provider

AWS can only check GitHub's tokens after you register GitHub as an **identity provider**. You do this once per AWS account.

1. In the AWS console, open **IAM → Identity providers**.
2. If `token.actions.githubusercontent.com` is already listed, open it and check that `sts.amazonaws.com` is one of its audiences. Then go to step 3.
3. Otherwise, click **Add provider** and fill in:
   - Provider type: **OpenID Connect**
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
4. Click **Add provider**.

Other projects in your AWS account may use the same provider, so it stays in place, with any other audiences it already has.

## Step 3: Create the role with the trust policy

1. Open **IAM → Roles** and click **Create role**.
2. Under **Trusted entity type**, choose **Custom trust policy**.
3. Delete the example policy in the editor, paste the whole contents of `build/github-oidc-trust.json`, and click **Next**.
4. On **Add permissions**, select nothing and click **Next**. The permissions come from the second file in step 4.
5. Name the role exactly as the dry run printed, such as `hr-ci-yourname-github-ci`, and click **Create role**.

## Step 4: Add the permissions as an inline policy

1. Open the new role. On its **Permissions** tab, click **Add permissions → Create inline policy**.
2. Switch the editor to **JSON**, delete the example, paste the whole contents of `build/github-oidc-permissions.json`, and click **Next**.
3. Name the policy `HRLabOnly` and click **Create policy**. Use exactly this name: it is `POLICY_NAME` in `scripts/prepare-oidc.py`, so a later run of the script updates this same policy instead of adding a second one.

## Step 5: Save the role ARN

The **ARN** is the role's full AWS name. On the role's summary page, copy it. Save it in the same file the script would use, then refresh the settings file for document 10:

```bash
cd ~/hr-rag-lab
echo "PASTE-THE-ROLE-ARN-HERE" > build/github-oidc-role-arn.txt
python scripts/github-settings.py
```

Replace `PASTE-THE-ROLE-ARN-HERE` with the ARN before you run it.

**Expected:** `GitHub settings: wrote build/github-settings.txt (15 variables, 2 secrets).`, with no `still missing` line. The counts are what the current pipeline needs.

## Check your work

Run the script with `--dry-run` again. It finds your role by its name, so it plans no new role:

```bash
python scripts/prepare-oidc.py --dry-run
```

**Expected:** the output includes `Role hr-ci-yourname-github-ci already exists: updating it`, with your role name, and ends with `Dry run: nothing was changed in AWS.`
