---
name: openworker-bedrock-setup
description: >-
  Download OpenWorker and wire it to the USER'S OWN AWS Bedrock (DeepSeek + Claude) with a
  least-privilege IAM setup. Hand-holds AWS account creation/sign-in, installs the AWS CLI,
  generates a scoped Bedrock key into a locked-down env file, helps enable Bedrock model
  access, then walks the user through configuring OpenWorker's native Bedrock provider,
  clicking Test, and adding the DeepSeek + Claude models. Use when a user wants OpenWorker
  running on their own Bedrock account. Read the repo README (pre-setup + safety) first.
---

# OpenWorker on the User's Own AWS Bedrock

You are setting up **OpenWorker** (Andrew Ng's local-first desktop agent) to run on the **user's own
AWS Bedrock**, with least-privilege access. Work **one step at a time**, confirm each before moving on,
and **stop and hand control to the user** for the steps that must be human. Never invent credentials,
never store a key anywhere but the locked env file, and never echo a secret back in plaintext.

## Guardrails (apply throughout)
- **Confirm the machine first.** Ask: "Is this a machine dedicated to the agent, separate from personal
  banking / 401(k) / password managers?" If no, point them to the README safety section and **do not
  proceed** until they confirm or explicitly accept the risk.
- **Least privilege only.** The agent identity gets the Bedrock policy from the README and nothing more.
- **Secrets discipline.** Keys go into `~/.openworker-agent/bedrock.env` (`chmod 600`), git-ignored,
  never printed in full, never committed, never pasted into a chat.
- **Human-only steps** (you cannot do these headlessly): creating an AWS account, approving Bedrock model
  access, and clicking buttons inside the OpenWorker desktop app. Prepare the exact clicks and pause.

## Step 1 — Download & install OpenWorker
1. Detect OS. Get OpenWorker from the official source: `https://github.com/andrewyng/openworker`
   (releases page for a packaged desktop build, or clone + `pip install 'openworker[bedrock]'` for the
   source path — the **`[bedrock]`** extra is required so boto3 is present).
2. Launch it once to confirm the local server comes up (it binds `127.0.0.1:8765`) and the Settings ▸ Models
   screen loads. Leave it running; you'll return to it in Step 7.

## Step 2 — AWS account (create or sign in)
- Ask: "Do you already have an AWS account you want the agent to use?"
- **If yes:** have them sign in. Strongly recommend a *dedicated* account (AWS Organizations, free) or at
  least a dedicated IAM user — see the README safety section.
- **If no:** you **cannot create it for them** (AWS requires email, a payment card, and phone verification —
  not automatable). Hand-hold them through `https://portal.aws.amazon.com/billing/signup`, then:
  - Enable **MFA on the root user**; do not create root access keys.
  - Come back once they can sign in to the Console.

## Step 3 — Install the AWS CLI
- **macOS:** `brew install awscli` (or the official pkg installer).
- **Linux:** `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip && unzip -q /tmp/awscliv2.zip -d /tmp && sudo /tmp/aws/install`
- Verify: `aws --version`.

## Step 4 — Create the least-privilege agent identity + policy
Run as an admin identity (the user's console or an admin profile), substituting a real account:
1. Save the policy from the README to `bedrock-least-priv.json`, then:
   ```
   aws iam create-policy --policy-name OpenWorkerBedrockLeastPriv \
     --policy-document file://bedrock-least-priv.json
   ```
2. Create a dedicated user with **no console access**:
   ```
   aws iam create-user --user-name openworker-agent
   aws iam attach-user-policy --user-name openworker-agent \
     --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/OpenWorkerBedrockLeastPriv
   ```
3. Confirm the user has **only** that policy attached (no `AdministratorAccess`, no billing).

## Step 5 — Generate a key the least-messy way
Prefer a **Bedrock API key** (bearer token) — the no-SigV4 path OpenWorker likes; it rides
`AWS_BEARER_TOKEN_BEDROCK`.
- **CLI (long-term Bedrock key = an IAM service-specific credential):**
  ```
  aws iam create-service-specific-credential \
    --user-name openworker-agent \
    --service-name bedrock.amazonaws.com
  ```
  Capture the returned key value **once** (it is shown only at creation).
- **Console fallback:** Bedrock Console ▸ **API keys** ▸ *Generate* (long-term), scoped to `openworker-agent`.
- If the user prefers classic IAM keys instead: `aws iam create-access-key --user-name openworker-agent`
  (then OpenWorker's Bedrock provider uses the **IAM keys** auth method rather than API-key).

### Store it locked down (give the agent access via env file)
```
mkdir -p ~/.openworker-agent && chmod 700 ~/.openworker-agent
umask 177
cat > ~/.openworker-agent/bedrock.env <<'EOF'
# OpenWorker / Bedrock — agent credentials. DO NOT COMMIT.
AWS_BEARER_TOKEN_BEDROCK=<PASTE_KEY_HERE>
AWS_REGION=us-west-2
EOF
chmod 600 ~/.openworker-agent/bedrock.env
```
Add `~/.openworker-agent/` (and `*.env`) to the machine's global gitignore. To load for CLI/agent use:
`set -a; source ~/.openworker-agent/bedrock.env; set +a`. **Never** print the file's contents in full.

## Step 6 — Enable Bedrock model access (mostly human)
Model access is a one-time approval, largely **console-only** (Anthropic models require a short use-case form):
1. AWS Console ▸ **Bedrock** ▸ **Model access** ▸ *Manage/Enable* — in the **region you'll use** (default
   **us-west-2**; it must match `AWS_REGION`).
2. Enable at minimum: **DeepSeek** (V3 / V3.2) and, for top capability, **Anthropic — Claude Sonnet 4.6**.
3. Wait for status **Access granted** (Claude may take a few minutes after the form).
4. Verify from the CLI (uses the agent key; proves the Test button will pass):
   ```
   curl -s "https://bedrock-runtime.us-west-2.amazonaws.com/openai/v1/chat/completions" \
     -H "Authorization: Bearer $AWS_BEARER_TOKEN_BEDROCK" -H "Content-Type: application/json" \
     -d '{"model":"deepseek.v3-v1:0","messages":[{"role":"user","content":"ping"}],"max_tokens":8}'
   ```
   A JSON completion back = key + region + model access all good.

## Step 7 — Configure OpenWorker's native Bedrock provider
Do this **inside the OpenWorker app** (guide the user click-by-click):
1. Settings ▸ **Models** ▸ **AWS Bedrock** (the native provider — **not** a custom OpenAI endpoint, **not**
   the public "DeepSeek" tile; that tile sends `deepseek-v4-flash`, which does not exist on Bedrock).
2. Fields:
   - **AWS region:** `us-west-2` (match Step 6).
   - **Connect with:** `Bedrock API key` → paste the key from Step 5. (Or `IAM keys` if you made access keys.)
3. Click **Test** — it should go **green**. It runs `ListFoundationModels`; a red result means the key or
   region is wrong, or the policy is missing `bedrock:ListFoundationModels`.

### Endpoint & key, for reference / external testing
- OpenWorker's native provider needs only **region + key** (no URL).
- If a raw OpenAI-compatible endpoint is ever needed: `https://bedrock-runtime.<region>.amazonaws.com/openai/v1`
  with the same Bedrock API key as the bearer token. (Note: that path's `/models` returns 404 — use the
  native provider in OpenWorker, which sidesteps it.)

## Step 8 — Add the models (verified against a live account, us-west-2)

In Settings ▸ Models ▸ **Add model**, paste the model **ID** — the friendly name ("DeepSeek V3.2") is a
label, not an ID. A model ID in OpenWorker is `<provider-prefix>:<bedrock-model-id>`, where the prefix is
the provider you configured in Step 7. Confirmed working full string: **`deepseek:deepseek.v3-v1:0`**.

Two routes reach Bedrock, and which one a model uses is fixed by the model family:

**A. Open-weight chat models** — the OpenAI-compatible / Converse route (the provider you point at Bedrock).
All verified runnable on this account:

| Model | Bedrock model ID |
|---|---|
| DeepSeek V3 (recommended default) | `deepseek.v3-v1:0` |
| DeepSeek V3.2 | `deepseek.v3.2` |
| Mistral Large 3 (675B) | `mistral.mistral-large-3-675b-instruct` |
| Qwen3 235B | `qwen.qwen3-235b-a22b-2507` |
| Qwen3 Coder 480B | `qwen.qwen3-coder-480b-a35b-instruct` |
| GLM-5 | `zai.glm-5` |
| Kimi K2.5 | `moonshotai.kimi-k2.5` |
| MiniMax M2.5 | `minimax.minimax-m2.5` |
| Nemotron Super 3 120B | `nvidia.nemotron-super-3-120b` |
| gpt-oss-120b | `openai.gpt-oss-120b` |

**B. Anthropic Claude** — OpenWorker's **native Anthropic-on-Bedrock** path (the Messages API, not
chat-completions). Available on this account:
- Claude Sonnet 4.6 — `anthropic.claude-sonnet-4-6-v1:0`
- Claude Haiku 4.5 — `anthropic.claude-haiku-4-5`

**NOT available without an AWS access request** (higher-tier models sit behind a Bedrock model-access /
use-case review wall): **DeepSeek R1, Claude Opus 4.8 / Opus 5, Claude Sonnet 5**. Also **GPT-5.x** (only on
the OpenAI Responses API, not chat-completions) and **Grok 4.3** (not exposed on the chat route). If you need
these, request access in Bedrock ▸ Model access and wait for approval.

The full available-model list for this account tier is in the repo README.

## Step 9 — Select & verify end-to-end
1. In the composer's model dropdown, **select** the model you added (adding ≠ selecting — the composer keeps
   running its previous default until you switch it).
2. Run a real prompt. Confirm the response and that usage shows on the model you chose (not `deepseek-v4-*`).
3. Remove any leftover **public DeepSeek** provider so nothing falls back to `deepseek-v4-flash`.

## Step 10 — Close out
- Confirm: budget alert set (README §2), env file is `600` + git-ignored, root MFA on, agent user has only
  the Bedrock policy.
- Tell the user their working defaults:
  - Everyday (open-weight route): `deepseek:deepseek.v3-v1:0`
  - Most capable available: Claude Sonnet 4.6 (`anthropic.claude-sonnet-4-6-v1:0`) via OpenWorker's native
    Anthropic-on-Bedrock integration — no extra approval needed on a standard account.
  - Gated (needs an AWS model-access request): DeepSeek R1, Claude Opus 4.8/5, Claude Sonnet 5.

## Failure decoder (quick)
- `model 'deepseek-v4-flash' does not exist` → the **public deepseek.com** tile is selected. Point the
  provider at your Bedrock endpoint and use a Bedrock model ID (e.g. `deepseek:deepseek.v3-v1:0`).
- `The provided model identifier is invalid` / `model doesn't exist` → wrong model ID, or the model is
  **gated** (R1 / Opus / Sonnet 5) and not approved on your account, or not enabled in that region.
- `model ... does not support the '/v1/chat/completions' API` → that model uses a different route
  (GPT-5.x → Responses API; Claude → the native Anthropic path). Add Claude via OpenWorker's Anthropic/Bedrock
  Claude integration, not the OpenAI-compatible provider.
- `AccessDenied … bedrock:ListFoundationModels` → policy too tight or wrong identity on the Test button.
- Test button red with a credentials error → key not saved, wrong auth method, or region mismatch.
