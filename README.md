# OpenWorker on Your Own AWS Bedrock — Agent Skill

An agent **skill** that sets up [OpenWorker](https://github.com/andrewyng/openworker) (a local-first AI
coworker) to run on **your own AWS Bedrock** — DeepSeek and Claude — with a **least-privilege** access
model. The agent downloads OpenWorker, hand-holds AWS sign-in, installs the AWS CLI, generates a scoped
Bedrock key into a locked-down env file, helps enable Bedrock model access, then walks you through wiring
OpenWorker's native Bedrock provider, clicking **Test**, and adding the exact working model IDs.

- **[`openworker-bedrock-setup.md`](./openworker-bedrock-setup.md)** — the skill itself (the instructions the agent follows).
- **This README** — read the **safety section first**, then pick how to run it.

> ⚠️ **Read [Before you run it — safety & least access](#before-you-run-it--safety--least-access) first.**
> This skill creates cloud credentials and runs commands on your machine. Where and how you run it matters
> more than the skill itself.

---

## How to run it

This is a plain-Markdown instruction file, so any coding agent that can read a file and run shell commands
can execute it. Two common ways:

### Claude Code (Anthropic)

Claude Code has native **skills** support. Install it as a skill and invoke by name:

```bash
git clone https://github.com/ElderberryAI/openworker-bedrock-setup.git
mkdir -p ~/.claude/skills/openworker-bedrock-setup
# Claude Code skills must be named SKILL.md inside a folder named after the skill:
cp openworker-bedrock-setup/openworker-bedrock-setup.md ~/.claude/skills/openworker-bedrock-setup/SKILL.md
```

Then in Claude Code:

```
/openworker-bedrock-setup
```

(or just ask: *"set up OpenWorker on my own AWS Bedrock"* — Claude Code will pick up the skill by its
description).

### OpenAI Codex CLI (the OpenAI equivalent)

OpenAI's equivalent coding agent is the **Codex CLI**. It doesn't have a `skills/` folder, so you hand it
the file as the task:

```bash
npm install -g @openai/codex          # install the Codex CLI
git clone https://github.com/ElderberryAI/openworker-bedrock-setup.git
cd openworker-bedrock-setup
codex "Follow the steps in openworker-bedrock-setup.md exactly, one at a time, pausing for the human-only steps."
```

Alternatively, drop the skill's contents into an `AGENTS.md` in your working directory and Codex will load
it as standing instructions.

### Any other agent

Point your agent at [`openworker-bedrock-setup.md`](./openworker-bedrock-setup.md) as its instructions.
The steps are tool-agnostic; the only hard requirement is that the agent can run shell commands and read
your responses (it deliberately **pauses** for the steps a human must do — AWS account creation, model-access
approval, and clicking buttons in the OpenWorker desktop app).

---

## What you'll end up with

OpenWorker running against **your own** Bedrock, billed to your AWS account, from a machine that holds none
of your personal life. In OpenWorker a model ID is `<provider-prefix>:<bedrock-model-id>` — the prefix is the
provider you configured. **Confirmed working:** `deepseek:deepseek.v3-v1:0`.

**The two models this repo sets up** (both verified working, us-west-2; cost is on-demand per 1M tokens):

| Model | OpenWorker string (paste verbatim) | Best for | Cost — in / out |
|---|---|---|---|
| DeepSeek V3 | `deepseek:deepseek.v3-v1:0` | General default — strong all-round, cheap | ~$0.62 / $1.85 |
| DeepSeek V3.2 | `deepseek:deepseek.v3.2` | Newest DeepSeek — best value general/agentic | $0.62 / $1.85 |

> The friendly name ("DeepSeek V3.2") is a **label, not an ID** — always paste the exact string with your
> provider prefix. If a model ID has no known prefix, OpenWorker silently routes it to OpenAI.
>
> Bedrock offers many other models (Claude, Qwen, GLM, Mistral, …) on the same account, but they're fussier
> to wire up. This repo keeps it deliberately simple to the two DeepSeek models.

> **Note:** higher-tier models (DeepSeek R1, Claude Opus 4.8 / Opus 5, Claude Sonnet 5) sit behind an AWS
> model-access approval wall and are **not** part of this quick-start. Request them in Bedrock ▸ Model access
> if you need them later.

---

## Before you run it — safety & least access

Read this **completely** first. It explains *where* to run an autonomous agent and *how* to scope its
access so a mistake (or a prompt-injection) can't reach anything that matters.

### 1. Use a machine dedicated to the agent — the #1 rule

Run OpenWorker (and any autonomous agent, **including Claude Code**) on a computer you treat as
**disposable and isolated**. The recommendation — even from the Claude Code team — is separation of access:
the agent can run commands, read files, and hold credentials, so you want its blast radius to contain
*nothing personal*.

**Do NOT run the agent on the same machine / OS-user you use for:**
- Banking, brokerage, your 401(k)/retirement accounts, tax software
- Personal email, password managers, 2FA seed vaults
- Signed-in browsers with saved cards, SSO to your employer, cloud consoles for *other* accounts
- Anything with production or customer data

**Do instead — cheapest first:**
- A separate **OS user account** with no access to your personal files (minimum bar), **or**
- A dedicated **VM** (UTM / VirtualBox / cloud VM) you can snapshot and wipe, **or**
- A separate physical machine (a cheap mini-PC or spare laptop).

Treat the agent machine as **hostile-tolerant**: assume anything it can read or reach could be exfiltrated,
and design so that "everything it can reach" is boring.

### 2. Least-access paradigm (defense in depth)

Grant the **smallest** access that still lets the job run. Each layer limits a different blast radius.

| Layer | Least-access rule |
|---|---|
| **Machine** | Dedicated box / VM / OS-user. No personal logins on it. Snapshot before first run. |
| **AWS account** | Use a **separate AWS account** for the agent if you can (AWS Organizations makes this free). At minimum a **dedicated IAM user**, never the root user. |
| **Root user** | Enable **MFA on the root user**, then never use it. No root access keys — delete any that exist. |
| **IAM identity** | A single IAM user `openworker-agent` with **only** the Bedrock policy below. No console login, no `iam:*`, no billing, no S3, nothing else. |
| **Credentials** | Prefer a **Bedrock API key** (bearer token) or short-lived STS creds over long-lived IAM access keys. Rotate on a schedule. |
| **Secrets at rest** | Key lives in one file, `chmod 600`, owned by the agent user, **git-ignored**. Never in a repo, never in shell history, never pasted into chats. |
| **Spend** | An **AWS Budget** with an email alert (e.g. $20/mo) + a hard alarm. Bedrock is pay-per-token; a runaway loop should page you, not surprise you. |
| **Network** | If the box only needs AWS + GitHub, consider egress allow-listing. Optional but cheap insurance. |

#### Least-privilege IAM policy (attach ONLY this to `openworker-agent`)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BedrockInvokeChosenModelsOnly",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream",
        "bedrock:Converse",
        "bedrock:ConverseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:*::foundation-model/deepseek.*",
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-*",
        "arn:aws:bedrock:*:*:inference-profile/us.deepseek.*",
        "arn:aws:bedrock:*:*:inference-profile/us.anthropic.*"
      ]
    },
    {
      "Sid": "BedrockListForOpenWorkerTestButton",
      "Effect": "Allow",
      "Action": [
        "bedrock:ListFoundationModels",
        "bedrock:GetFoundationModelAvailability"
      ],
      "Resource": "*"
    }
  ]
}
```

This grants **invoke on the two model families you'll use and nothing else**, plus the read-only
`ListFoundationModels` that OpenWorker's **Test** button calls. It cannot touch IAM, billing, S3, or any
other service. Tighten `Resource` further (drop `anthropic.*` or `deepseek.*`) if you only want one vendor.

> Enabling **model access** (a separate, one-time console step) is intentionally **not** in this policy —
> do it once as an admin, then the agent user only ever *invokes*.

### 3. Pre-setup checklist (do these, then run the skill)

- [ ] Agent runs on a **dedicated machine / VM / OS-user** (§1).
- [ ] You have (or will create) an AWS account you're comfortable being the agent's sandbox.
- [ ] **Root user has MFA**; no root access keys exist.
- [ ] You can reach the **AWS Console** in a browser to enable Bedrock model access + click a couple buttons.
- [ ] A payment method + phone are on hand **only if** you still need to create the AWS account (that part is manual — AWS requires it and it cannot be automated).
- [ ] You accept Bedrock is **pay-per-token** and will set a budget alert during setup.

When all boxes are checked, run the skill. It pauses and hands control back to you for the few steps that
*must* be human (account creation, model-access approval, clicking **Test** in OpenWorker).

---

## License

MIT. Provided as-is; you are responsible for your own AWS account, spend, and credential hygiene.
