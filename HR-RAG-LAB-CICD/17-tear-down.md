# 17 Tear down

**Goal:** delete everything the lab created in AWS, so you stop paying for it.

Some parts cost money every hour until you delete them, even when nobody uses the app. These are the private network connections, the model gateway server and the web firewall (WAF).

Tear down only after your last use case, and after you saved your evidence links. Run it on the same computer that ran the setup. Terraform, the tool that built the AWS resources, can only delete what it recorded on that computer.

After the teardown, your GitHub runs, Langfuse traces and pull requests are still there. The AWS resources, including the `approved` alias, are gone.

## Step 1: Stop new deployments

In GitHub, open **Actions → HR application delivery and evaluation → ⋯ → Disable workflow**.

If you skip this, a merge to `main` would start a run that tries to deploy into resources that no longer exist.

## Step 2: Run the teardown script

```bash
cd ~/hr-rag-lab
python scripts/tear-down.py
```

The script does the rest, in this order:

1. Reads your project's names from Terraform.
2. Runs `terraform destroy`. It lists everything it will delete and asks you to type `yes`. Check that every name starts with your own `project_name`, then type `yes`.
3. Deletes the pipeline's AWS role from document 06. Terraform did not create that role, so it cannot delete it.
4. Deletes `demo-sign-in.txt`, `.env.langfuse` and `build/github-settings.txt` from your folder. These files held passwords and keys.
5. Lists anything in AWS that is still labelled with your project.

Two options you can add to the command:

- `--release-names`: use it if you will set up the lab again with the same `project_name`. Two secrets hold the test accounts' passwords and the Langfuse keys. They use AWS's default recovery window: a deleted secret can be restored for 30 days, and its name stays taken. This option deletes them for good at once.
- `--dry-run`: shows what the script would delete, without deleting anything.

**Expected:** `Deleted the role`, then `Nothing tagged Project=` followed by your project name and `is left in AWS.`

If `terraform destroy` stops with a dependency error, run the script again. It carries on from what is left.

AWS can take a while to update its list of labelled resources. So if the script still lists something, open it in the AWS console to check whether it really still exists.

The script keeps the GitHub identity provider, the AWS setting that lets GitHub sign in. Other projects in your AWS account may use it.

## Step 3: Revoke the Langfuse keys

1. In Langfuse, open your project's **Settings → API keys** and revoke the key pair you used in this lab.
2. In GitHub, open **Settings → Environments → hr-staging** and delete the `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY` secrets.

The next day, open **AWS Billing** to check that the charges have stopped.
