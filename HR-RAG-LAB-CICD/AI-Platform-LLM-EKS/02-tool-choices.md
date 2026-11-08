# 2. The tool choices

Each part of Astra could be built with several tools. For each part, this document lists the options and the one this build uses. Then it shows the places on AWS where we could run the model.

---

## A. Build and deploy

### Infrastructure as code

- Options: Terraform, Pulumi.
- ✅ Pick: **Terraform**.

### GitOps

- Options: Argo CD, Flux.
- ✅ Pick: **Argo CD**, with Helm charts, set up through the GitOps Bridge.

### Image registry

- Options: Amazon ECR, Docker Hub, GitHub Container Registry.
- ✅ Pick: **Amazon ECR**, two private repositories in the same AWS account (`terraform/foundations/ecr.tf`): `astra/litellm` for the gateway image and `astra/k8s-assistant` for the app image.

---

## B. The model and the GPU

### The model

- Options: open-weight models such as Qwen3.8-27B, gpt-oss-20b, Mistral Small 3.2 24B, DeepSeek-R1-Distill-Qwen-32B, Gemma 3 27B, gpt-oss-120b and Llama 3.3 70B.
- ✅ Pick: **Qwen3.8-27B**, in FP8 (one byte per number), as `Qwen/Qwen3.8-27B-FP8`. How we narrow the list down: `05-model-pipeline`, section C.

### The GPU machine

- Options: `g5` (NVIDIA A10G, 24 GB) and `g6` (NVIDIA L4, 24 GB): too small for the FP8 model. `g6e` (NVIDIA L40S, 48 GB). `p4d` and `p5` (8 large GPUs per machine): far more than one model replica needs.
- ✅ Pick: **`g6e.xlarge`** (one L40S, 48 GB), the smallest machine where the FP8 model fits one GPU. The NodePool also allows `g6e.2xlarge` and `g6e.4xlarge` (still one GPU), for when AWS has no `g6e.xlarge` free (`04-platform-addons`).

---

## C. GPU nodes and serving

### GPU nodes

- Options: Karpenter, Cluster Autoscaler, EKS Auto Mode.
- ✅ Pick: **Karpenter**.

### Inference engine

(**Inference engine** is another name for the inference server.)

- Options: vLLM, SGLang, Hugging Face TGI (Text Generation Inference), NVIDIA TensorRT-LLM with the Triton Inference Server (the most speed from NVIDIA GPUs, with more build work), Ollama or llama.cpp (for one person on a laptop, not for a service with many users).
- ✅ Pick: **vLLM**.

### Deploying vLLM

- Options: our own Helm chart, KServe (a Kubernetes add-on that deploys and scales models through its own objects).
- ✅ Pick: **our own Helm chart**.

### Model loading

- Options: copy the model to a disk first, or stream it from S3.
- ✅ Pick: **S3 with the Run:ai Model Streamer**.

### Pod autoscaling

- Options: KEDA, the built-in Horizontal Pod Autoscaler.
- ✅ Pick: **KEDA**, scaling on the vLLM queue (the number of requests waiting for a GPU).

---

## D. Model pipeline

This build runs the first part of a model pipeline by hand: the loader Job downloads, scans and uploads one model version, then registers it (`05-model-pipeline`). In production, a full pipeline adds an **eval gate** (tests that run on a model and stop it if the scores are too low) and an approval. The tools for those parts are below.

### Model scanning (built)

- Options: ModelScan (an open-source tool that scans model files for hidden code), the scans on the Hugging Face Hub, JFrog Xray, OpenSSF model signing (a tool that signs model files, so you can prove who made them and that nothing changed).
- ✅ Pick: **ModelScan**, and only **safetensors** files (a safetensors file holds only numbers and cannot run code). The loader Job runs it before the upload.

### Model registry (built)

- Options: MLflow, SageMaker Model Registry, Weights & Biases (a paid product that tracks ML experiments and models).
- ✅ Pick: **MLflow**, the same MLflow App as the traces. The loader Job registers each version it copies to S3.

### Pipeline tool (production, not built)

- Options: the CI/CD tools GitHub Actions, GitLab CI and Jenkins; Airflow; SageMaker Pipelines; Argo Workflows; Kubeflow Pipelines (a Kubernetes add-on that runs ML pipeline steps as pods).

### Eval gate (promptfoo built, the model gate explained)

- Options: lm-evaluation-harness (an open-source test runner that sends test questions to a model and scores the answers), promptfoo (a tool that sends test prompts to an app and checks the answers), others.

---

## E. Gateway and safety

### Gateway

- Options: LiteLLM, other open-source gateways, a cloud gateway.
- ✅ Pick: **LiteLLM**.

### Guardrails

- Options: guardrails inside LiteLLM, the Bedrock guardrail, other guardrail products.
- ✅ Pick: **LiteLLM guardrails** with **Presidio** and the **Bedrock ApplyGuardrail** API.

### Secrets

- Options: External Secrets with AWS Secrets Manager, the Secrets Store CSI driver, secrets in Git.
- ✅ Pick: **External Secrets** with **AWS Secrets Manager**.

### Policy

- Options: Kyverno, OPA Gatekeeper.
- ✅ Pick: **Kyverno**.

### Access

- Options: an internal ALB behind a VPN or zero-trust access, `kubectl port-forward`, SSM Session Manager.
- ✅ Pick: **an internal ALB behind a VPN or zero-trust access** in production. In this build: `kubectl port-forward`.

---

## F. Observability

### Metrics

- Options: kube-prometheus-stack, a managed Prometheus and Grafana, a paid monitoring product.
- ✅ Pick: **kube-prometheus-stack** with the NVIDIA DCGM exporter.

### LLM tracing

- Options: MLflow (managed on SageMaker AI, or self-hosted), Langfuse (an open-source LLM tracing tool, self-hosted or as the paid Langfuse Cloud), other LLM tracing tools.
- ✅ Pick: **MLflow on Amazon SageMaker AI**, as a serverless MLflow App in our own AWS account (`terraform/foundations/mlflow.tf`). The gateway sends one trace per request to it (`07-gateway`).
  - 🔒 The traces stay inside the account. They hold our company's text, so this matters more than anything else here.

---

## G. Where to run the model

Our condition is that prompts stay in our own AWS account (`00-why-own-inference`). Three AWS options meet it. This build uses the first one.

### EKS, self-hosted (this build)

- vLLM runs on GPU nodes in our own EKS cluster, from a Helm chart, and Argo CD keeps it in sync with Git.
- ✅ 🎛️ Full control: the vLLM version and flags, the model format, the GPU type, the zones, the scaling rule and the rollout.
- ✅ Any model that vLLM can run, the day vLLM supports it.
- ✅ The gateway, the guardrails and the apps run in the same cluster, with the same tools, and the traces stay in the same account.
- ⚠️ The platform team runs everything: the cluster, the nodes, the GPU drivers, the upgrades and the on-call.
- ⚠️ 💰 GPUs cost the same every hour, busy or not. EC2 quota and free capacity are needed (`00-why-own-inference`, section F).
- Pick it when you already run Kubernetes, want one platform for many models and apps, and have a platform team to run it.

### A SageMaker AI real-time endpoint

(An **endpoint** here is a model that AWS hosts on instances it manages, and that apps call over an AWS API.)

- AWS offers the LMI container (Large Model Inference). Its current image is based on vLLM, with AWS performance patches ([LMI release notes](https://docs.djl.ai/master/docs/serving/serving/docs/lmi/release_notes.html)).
- ✅ AWS runs the instances. Auto scaling adds and removes instances as the load changes ([Automatic scaling](https://docs.aws.amazon.com/sagemaker/latest/dg/endpoint-auto-scaling.html)).
- ✅ Deployment guardrails: an update is a blue/green deployment, with all at once, canary or linear traffic shifting. If a CloudWatch alarm fires, SageMaker rolls the traffic back by itself ([Blue/Green deployments](https://docs.aws.amazon.com/sagemaker/latest/dg/deployment-guardrails-blue-green.html)).
- ⚠️ 💰 During a canary or linear update, you pay for both fleets for the whole update (same page).
- ⚠️ SageMaker has its own instance quotas, separate from the EC2 quota. For example, **"ml.g6e.xlarge for endpoint usage"** has a default of 0 in each region, and can be raised ([SageMaker AI quotas](https://docs.aws.amazon.com/general/latest/gr/sagemaker.html)).
- ⚠️ Apps call it with `InvokeEndpoint`, signed with AWS Signature Version 4, not with an OpenAI-style API key. The model container must answer within 60 seconds ([InvokeEndpoint](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_runtime_InvokeEndpoint.html)).
- ⚠️ So a gateway such as LiteLLM is still needed in front of it: for team API keys, budgets, an OpenAI-style API and usage per team.
- Pick it when you already use SageMaker, want AWS to run the instances and the rollout, and have no team to run Kubernetes.

### Amazon Bedrock Custom Model Import

- You import the model weights from S3 (safetensors and the Hugging Face config and tokenizer files; the **tokenizer** splits text into tokens), and Bedrock serves them. Apps call `InvokeModel`, on demand ([Custom Model Import](https://docs.aws.amazon.com/bedrock/latest/userguide/model-customization-import-model.html)).
- ✅ No instances, nodes or inference server to run.
- ⚠️ Only listed architectures (model designs, named in the model's `config.json`) can be imported. For Qwen3, only `Qwen3ForCausalLM` and `Qwen3MoeForCausalLM` are listed (same page).
- ⚠️ This build's model, `Qwen/Qwen3.8-27B-FP8`, names the architecture `Qwen3_5ForConditionalGeneration` in its `config.json`, and its context length (the most tokens in one request, prompt plus answer) is 262,144 tokens. The page says the context must be less than 128K. So this model cannot be imported as it is today.
- ⚠️ Other limits on the same page: text model weights must be under 200 GB, the model must work with transformers 4.51.3 (the Hugging Face Python library), it is offered in four regions (us-east-1, us-east-2, us-west-2, eu-central-1), and batch inference (many prompts sent as one job) is not supported.
- ⚠️ A gateway is still needed for team API keys, budgets and usage per team.
- Pick it when the model's architecture is on the list and you want no infrastructure at all.
