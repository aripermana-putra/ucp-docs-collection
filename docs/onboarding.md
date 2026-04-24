---
title: "#3-0. Getting Started — New Joiner Guide"
space: UCP
parent_page_id: "6611029491"
---

## Welcome to the UCP Group

Welcome aboard. This guide covers everything you need to get set up before you start diving into the codebase and day-to-day work. Work through this during your first week — the earlier you raise access requests, the better, as some provisioning can take time.

For the next step after completing this guide, see [Getting Started with the Codebase and Project](<https://pages.ghe.rakuten-it.com/clsd-ucp/ucp-developer-guide/getting-started/ONBOARDING/>).

---

## 1. About the Team

**Universal Control Plane (UCP)** is a platform team working toward a unified control plane that can provision, manage, and monitor services across Rakuten cloud environments — both public and private cloud.

We sit within the organization as follows:

```
CLSD (Cloud Services Department)
  └── UCP Section  (L3: Kimura-san)
        └── UCP Group  (L4: Kimura-san)
```

The group is still in its early days, so things will evolve — expect some ambiguity and bring your initiative.

### 1.1 Meet the Team

| Name | Nickname | Role |
| --- | --- | --- |
| Ryo Kimura | Ryo | L3 and L4 Manager |
| Sebastien Bellefeuille | Sebas | Vice Manager / Architect |
| Yusuke Ohashi | Mike | Senior Software Engineer |
| Ari Permana Putra | Ari | Senior Software Engineer |
| Rania Ben Kahla | Rania | Product Manager |

---

## 2. Access Requests

You should start doing this after you're done with the basic setup for your device. At this point, you should already have your Rakuten SSO account and Office 365 (for your email) all ready to go.

> Start these as early as possible — approvals and provisioning can take time.

### 2.1 VPN (DEV-VPN)

> VPN (DEV-VPN) is the first thing to set up — most of the dev environment is only accessible through it.

> Your VPN account should already be provisioned when you join (Check your email inbox with subject **[DEV-VPN] We created your account** and **[DEV-VPN REMOTE] We created your account**). If it has not been set up, raise a request following this guide: [[DEV-VPN] How to apply for DEV-VPN](<https://rakutenghd.zendesk.com/hc/en-us/articles/215507598--DEV-VPN-How-to-apply-for-DEV-VPN>).

**Prerequisite:** **Cisco Secure Client** (formerly AnyConnect) should already be installed as part of your device's initial setup. If it is not, refer to: [Remote Access (R-VPN) — How to install and update R-VPN client](https://rakutenghd.zendesk.com/hc/en-us/articles/360001355247--Remote-Access-R-VPN-How-to-install-and-update-R-VPN-client)

For the full DEV VPN setup walkthrough, refer to: [DEV VPN — How to access DEV VPN](https://rakutenghd.zendesk.com/hc/en-us/articles/360022655254--DEV-VPN-How-to-access-DEV-VPN)

Once you have your account, verify that VPN works in both scenarios:

| Scenario | Description |
| --- | --- |
| Intra (in-office) | Connect to DEV-VPN-INTRA (p-dev-intra.r-vpn.net) while on the office network (**r-intra**) |
| Remote (WFH) | Connect to DEV-VPN-REMOTE (p-dev-remote.r-vpn.net) environment from outside the office (or connect to **r-byod** while in the office) |

> DEV-VPN-REMOTE works as well if you are connected to **r-intra**, so you can just connect to it instead all the time, whether you are in the office or outside

### 2.2 Slack

Although Rakuten uses Microsoft Teams, the UCP team (and much of CLSD) communicates via **R-Slack** — Rakuten's internal Slack workspace.

**How to request access:**

1. Follow the guidance in this document to request access: [[Read First] R-Slack — About R-Slack](https://rakutenghd.zendesk.com/hc/en-us/articles/360016402174--Read-First-R-Slack-About-R-Slack)
2. Log in using your Rakuten SSO account
3. Kimura-san (L3 manager) will receive a notification to approve the request
4. After approval, wait for the account creation process to complete — check your approval/creation status here: [R-Slack Account Status](https://apps.powerapps.com/play/e/default-53a8b0d9-d900-48cc-9d7e-5935dc8d5b17/a/d4f07dbd-9d3c-4168-bf77-74f14609ea27)

Once your account is ready, access R-Slack at [rakuten.slack.com](https://rakuten.slack.com/) — you can also install the Slack app on your mobile device and log in there.

See section 3.2 for the list of channels to join.

### 2.3 Rakuten AI Gateway

Rakuten AI Gateway is the internal platform for accessing AI-powered developer tools, including the agentic coding assistants used by the team.

- **Web portal:** [Rakuten AI Gateway](https://developer.ai.public.rakuten-it.com/overview/0ad12720-bcf3-4214-929a-cd5d57c114e3)
- **Documentation:** [Rakuten AI Gateway Docs](https://pages.ghe.rakuten-it.com/AI4B/rakuten-ai-gateway-docs/index.html)

**How to request access:**

1. Go to the [User Onboarding guide](https://pages.ghe.rakuten-it.com/AI4B/rakuten-ai-gateway-docs/coding-agent/user-onboarding/user-onboarding.html) and follow **Case 2**
2. Use SID **100553** when prompted
3. Notify Kimura-san (L3 manager) to approve the request

Once approved, you will have access to the following tools:

| Tool | Type | Notes |
| --- | --- | --- |
| Claude Code | Agentic coding assistant (CLI) | By Anthropic — see section 3.3 for setup |
| Codex | Agentic coding assistant | By OpenAI — see section 3.3 for setup |

### 2.4 GitHub Enterprise (GHE)

Rakuten uses several Git-based repository managers (Bitbucket, GitHub Cloud, GitHub Enterprise). The UCP team uses **GitHub Enterprise (GHE)** — a self-hosted GitHub instance at [ghe.rakuten-it.com](https://ghe.rakuten-it.com).

**How to get access:**

1. Sign up at [github.com](https://github.com) using your Rakuten email address if you do not have a GitHub account yet
2. You can log in to GHE with the same GitHub account — however, without an explicit access request, your account will be suspended after some time
3. Follow the request guide in this document: [GitHub Enterprise — About GitHub Enterprise and Copilot](https://rakutenghd.zendesk.com/hc/en-us/articles/26524830883353--GitHub-Enterprise-About-GitHub-Enterprise-and-Copilot)
4. After filling out the form, Kimura-san (L3 manager) will receive a notification to approve
5. After approval, processing takes some time — raise this request as early as possible

Once your GHE access is active, ask a senior team member to add you to the UCP repositories. You can browse the full list of repositories at [ghe.rakuten-it.com/clsd-ucp](https://ghe.rakuten-it.com/clsd-ucp).

### 2.5 Jira & Confluence

Rakuten uses self-hosted Jira and Confluence instances. You should already be able to log in to both using your Rakuten intra SSO account.

| Tool | URL |
| --- | --- |
| Jira | [jira.rakuten-it.com](https://jira.rakuten-it.com) |
| Confluence | [confluence.rakuten-it.com](https://confluence.rakuten-it.com) |

Ask a senior team member to add you to the following:

| What | Link |
| --- | --- |
| UCP Jira project | [MCUCP Board](https://jira.rakuten-it.com/jira/secure/RapidBoard.jspa?projectKey=MCUCP&rapidView=46297&view=planning) |
| UCP Confluence space | [Universal Control Plane Home](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6227001101/Universal+Control+Plane+Home) |

### 2.6 Cloud Platform Access

Ask a senior team member to add you to the platforms below.

**GCP (Google Cloud Platform)**

Sign up and sign in at [console.cloud.google.com](https://console.cloud.google.com) using your Rakuten email account. Then ask a senior team member to add you to the UCP sandbox project:

- **Project:** `sub-gcp-ucp-clsd-sandbox`
- **Console:** [console.cloud.google.com/welcome?project=sub-gcp-ucp-clsd-sandbox](https://console.cloud.google.com/welcome?project=sub-gcp-ucp-clsd-sandbox)

**OneCloud (Private Cloud)**

OneCloud is Rakuten's internal private cloud platform. Log in to the QA environment using your Rakuten intra SSO account (Okta), then ask a senior team member to add you to the UCP tenant:

- **URL:** [qa-portal-onecloud.rakuten-it.com](https://qa-portal-onecloud.rakuten-it.com/)
- **Tenant:** `clsd-ucp`

### 2.7 Mail Distribution List

Join the UCP mailing lists via [dlm-plus.rakuten-it.com](https://dlm-plus.rakuten-it.com). Search for each group below and request to join:

| Distribution List | Scope |
| --- | --- |
| `clsd-ucp-group` | UCP Group |
| `clsd-ucp-section` | UCP Section |

### 2.8 (Optional) Connect Your Personal Device to r-byod

If you want to use your personal device in the office (e.g., your phone), you can connect it to the **r-byod** network.

Refer to this guide for setup instructions: [Wi-Fi Network for Employee's Private Devices (r-byod)](https://rakutenghd.zendesk.com/hc/en-us/articles/360009049593--Wi-Fi-Network-for-employee-s-private-devices-r-byod)

---

## 3. Tools & Setup

### 3.1 IDE

You are free to use any IDE you prefer. However, since the UCP repositories use dev containers, setup will be easiest with **VS Code** — it has native dev container support out of the box.

- **Download VS Code:** [code.visualstudio.com/download](https://code.visualstudio.com/download)

### 3.2 Slack

Join the following Slack channels once you have access:

| Channel | Purpose |
| --- | --- |
| `clsd-ucp-dev` | Main channel for the UCP group |
| `ucp-dbaas-omnia` | Communication channel with the DBaaS team |

> Ask a teammate to add you if you cannot find a channel.

### 3.3 AI Tooling

Once your Rakuten AI Gateway access is approved (see section 2.3), set up your chosen tool:

**Claude Code**

Follow the [Claude Code Setup Guide](https://pages.ghe.rakuten-it.com/AI4B/rakuten-ai-gateway-docs/coding-agent/claude-code/setup-guide.html).

> **Notes:**
> - If the installation via `curl` fails, install via Homebrew (Mac) or npm instead — refer to the [official Claude Code setup docs](https://code.claude.com/docs/en/setup#homebrew)
> - The Claude Code **desktop app** does not support the Rakuten AI Gateway API key — use the **CLI only**

**Codex**

Follow the [Codex Setup Guide](https://pages.ghe.rakuten-it.com/AI4B/rakuten-ai-gateway-docs/coding-agent/codex/setup-guide.html).

> The Codex desktop app can also be used — the setup guide covers how to inject the API key into it.

**Chat-based AI**

If you need an AI chat interface, two options are available:

- [Rakuten AI Portal](https://r-ai.tsd.public.rakuten-it.com/en-US/chats) — no setup needed, just log in with your Rakuten SSO (independent of Rakuten AI Gateway)
- Codex desktop app — requires completing the Codex setup above to configure the API key

---

## 4. Team Rituals & Communication

### 4.1 Daily Huddle

The team runs a daily huddle to sync on progress and blockers.

- **When:** Every day at 10:00 AM (unless rescheduled)
- **Where:** Zoom Meeting

Ask Kimura-san to add you to the calendar invitation.


### 4.2 Sprint Rituals

Sprint-based ceremonies (planning, retro, review) are planned and will be introduced as the group matures. Details will be shared when they are established.

### 4.3 Day-to-Day Communication

- Primary communication is via **Slack**
- The team communicates in **English**
- For questions on access or tooling, reach out to your manager/mentor or a fellow team member directly on Slack

---

## 5. First Week Checklist

Work through this list during your first week. Raise access request early so provisioning does not block your other setup steps.

| # | Item | How | Who to ask if stuck | Done |
| --- | --- | --- | --- | --- |
| 1 | Verify Cisco Secure Client is installed and VPN works (intra + remote) — request account if not provisioned | See section 2.1 | Mentor or Senior team member | [ ] |
| 2 | Request Slack workspace access | See section 2.2 | L3 Manager | [ ] |
| 3 | Raise Rakuten AI Gateway join request | See section 2.3 | L3 Manager | [ ] |
| 4 | Raise GHE org and repo access request | Jira ticket | L3 Manager for Approval. Mentor or Senior team member for repo access | [ ] |
| 5 | Raise Jira project access request | Jira ticket | Mentor or Senior team member | [ ] |
| 6 | Raise Confluence UCP space access request | Jira ticket | Mentor or Senior team member | [ ] |
| 7 | Get added to GCP sandbox project (sub-gcp-ucp-clsd-sandbox) | Mentor or Senior team member | Senior team member | [ ] |
| 8 | Get added to OneCloud UCP tenant (clsd-ucp) | Ask senior team member | Senior team member | [ ] |
| 9 | Join mail distribution lists (clsd-ucp-group, clsd-ucp-section) | See section 2.7 | Sebas or Kimura-san | [ ] |
| 10 | (Optional) Connect personal device to r-byod | See section 2.8 | Any teammate | [ ] |
| 11 | Set up IDE | See section 3.1 | Any teammate | [ ] |
| 12 | Join Slack channels | See section 3.2 | Any teammate | [ ] |
| 13 | Set up AI tooling (Claude Code / Codex) | See section 3.3 | Any teammate | [ ] |
| 14 | Attend daily huddle | See section 4.1 | Kimura-san | [ ] |

---

## 6. Next Steps

Once your access is in place and your tools are set up, you are ready to get into the work.

Proceed to: [Getting Started with the Codebase and Project](<https://pages.ghe.rakuten-it.com/clsd-ucp/ucp-developer-guide/getting-started/ONBOARDING/>)
That guide covers the repository structure, local development setup, key services, and how to make your first contribution.
