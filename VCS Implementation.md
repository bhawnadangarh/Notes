# Sprint-2 Interview & Implementation Handbook
## Part 1 of N — Epic: VCS Implementation

> **Series note:** This handbook is being produced epic-by-epic so each ticket gets genuine depth instead of shallow repetition. Part 1 covers the entire **VCS Implementation** epic (VCS setup, repos, authn, authz, commit/PR workflow, notifications, pre-commit hooks). Part 2 onward will cover Application CI Design, Cloud Infra Design, Cost Optimization, and Ansible — tell me the order you want, or I'll proceed sequentially.

---

# TICKET 1: Setup VCS

## 1. Introduction

**What is it?**
Version Control System (VCS) setup is the decision and configuration of the source-of-truth platform that stores every line of code, its full history, and the metadata (branches, tags, commits) needed to collaborate safely. In enterprise terms, this ticket is not "install Git" — Git is the protocol/tool everyone already assumes. The ticket is about **which hosting/collaboration layer sits on top of Git**: GitHub, GitLab, Bitbucket, or a self-hosted equivalent (Gitea, on-prem GitLab).

**Why do companies need it?**
Without a centralized VCS platform: no audit trail of who changed what, no enforced review process, no integration hooks for CI, no access control at repo/branch level, and no disaster recovery for source code — arguably a company's single most valuable asset.

**Why was this ticket created?**
It was flagged as a recommendation at the end of Sprint-1 (architecture/tool selection sprint). Sprint-2's job is to convert that recommendation into a running system with a live demo — proving the decision was correct and operational, not just theoretical.

**Real production example**
Netflix and Google run heavily customized internal Git hosting fused with Perforce/Piper (Google) for monorepo-scale needs; most mid-size enterprises (Adobe, most fintechs) standardize on GitHub Enterprise Cloud or GitLab Self-Managed specifically for the built-in CI, security scanning, and SSO integration.

**Where is it used?**
Every single engineering ticket downstream depends on this — CI pipelines, deployment automation, code review, compliance evidence, and audit trails all key off the VCS platform.

**What business problem does it solve?**
Traceability (who did what, when, why), collaboration at scale (parallel branches, PRs), risk reduction (nothing ships without review), and business continuity (code isn't sitting on one laptop).

---

## 2. Architecture

```
        ┌────────────────────┐
        │     Developer      │
        │ (local workstation)│
        └─────────┬──────────┘
                  │ git push / git pull
                  ▼
        ┌────────────────────┐
        │   Local Git Repo    │  ← .git/ object database (blobs, trees, commits)
        └─────────┬──────────┘
                  │ HTTPS/SSH
                  ▼
        ┌────────────────────┐
        │  Remote VCS Server  │  (GitHub/GitLab/Bitbucket)
        │  - Repo storage     │
        │  - Auth (SSO/PAT)   │
        │  - Webhooks         │
        │  - PR/MR engine     │
        └─────────┬──────────┘
                  │ Webhook (JSON payload on push/PR event)
                  ▼
        ┌────────────────────┐
        │   Jenkins / CI      │
        └────────────────────┘
```

**Component explanation:**
- **Local Git repo**: a content-addressable object store on disk (`.git/objects`) — every commit, tree (directory snapshot), and blob (file content) is stored as a SHA-1/SHA-256-hashed object.
- **Remote VCS Server**: adds the layer Git itself doesn't provide — user management, permissions, a web UI, PR/MR workflow engine, and webhook dispatch.
- **Webhook**: an HTTP POST the VCS server fires to a configured URL (Jenkins' `/github-webhook/` endpoint) whenever a specified event occurs (push, PR opened, comment added).
- **Auth layer**: SSO (SAML/OIDC) federating identity from the corporate IdP (Okta/Azure AD) so there's one identity source instead of per-tool passwords.

**SaaS vs Local (On-Prem) decision (interview-critical):**

| Factor | SaaS (GitHub.com / GitLab.com) | On-Prem (GitHub Enterprise Server / GitLab Self-Managed) |
|---|---|---|
| Setup time | Minutes | Days–weeks (infra, HA, patching) |
| Maintenance burden | None (vendor-managed) | Full ops burden on your team |
| Data residency/compliance | Data leaves your network | Full control, meets strict regulatory needs (banking, gov) |
| Cost model | Per-seat subscription | Infra + licensing + ops headcount |
| Customization | Limited to vendor features | Full control over plugins, network policy |
| Uptime/HA | Vendor SLA (typically 99.9%+) | You own HA/DR design |
| Air-gapped environments | Not possible | Only viable option |

**Recommendation logic to state in the demo:** for a growing product team without extreme regulatory constraints, SaaS (GitHub Enterprise Cloud) wins on velocity and reduced ops overhead; for regulated industries (banking, defense, healthcare with PHI) on-prem is often mandated regardless of convenience.

---

## 3. Internal Working

**What happens when you run `git push`?**
1. Git walks the local commit graph to determine which objects the remote doesn't have (a negotiation using "have"/"want" lines over the Git smart-HTTP or SSH protocol).
2. It packs the missing objects into a single **packfile** (delta-compressed) rather than sending each object individually — critical for performance on large repos.
3. The remote's Git process unpacks, validates (e.g., branch protection rules, signature requirements), and updates the ref (e.g., `refs/heads/main`) to point to the new commit SHA.
4. If branch protection/PR-only rules are enabled, a *direct* push to `main` is rejected by the server-side pre-receive hook — this is exactly the mechanism behind "only reviewers can merge to main."

**How does the webhook mechanism work internally?**
The VCS server maintains a list of registered webhook URLs per repo/event type. On a qualifying event (e.g., `push`, `pull_request`), the server's event bus serializes the event into a JSON payload matching a documented schema, signs it (HMAC using a shared secret, e.g., `X-Hub-Signature-256` in GitHub) and issues an HTTP POST. The receiver (Jenkins) validates the signature before trusting the payload — this prevents payload spoofing.

**How does SSO/SAML integration work under the hood?**
1. User hits the VCS login page → redirected to IdP (Okta) login (SP-initiated SAML flow).
2. IdP authenticates the user (possibly with MFA) and returns a signed SAML assertion (XML) to the VCS server's Assertion Consumer Service (ACS) URL.
3. VCS server validates the assertion's signature against the IdP's public certificate, extracts claims (email, groups), and either creates or maps to an existing user account (JIT provisioning).
4. Group claims can auto-map to VCS teams (e.g., `okta-group:devops` → GitHub team `devops-engineers`), which is how "no manual access provisioning" is achieved.

---

## 4. Tool Comparison — GitHub vs GitLab vs Bitbucket

| Criteria | GitHub | GitLab | Bitbucket |
|---|---|---|---|
| Built-in CI | GitHub Actions | GitLab CI/CD (very mature, native) | Bitbucket Pipelines |
| Built-in security scanning | Advanced Security (paid) | Ultimate tier (SAST/DAST/dependency scanning built-in) | Limited, relies on 3rd party apps |
| Self-hosted option | GitHub Enterprise Server | GitLab Self-Managed (very strong) | Bitbucket Data Center |
| Market ecosystem | Largest OSS ecosystem, most integrations | Strong all-in-one DevOps platform story | Best if already on Jira/Atlassian stack |
| Pricing model | Per seat, tiered | Per seat, tiered (Free/Premium/Ultimate) | Per seat, cheap at small scale |
| Enterprise adoption | Very high (esp. after Microsoft acquisition) | High in regulated/on-prem-first orgs | High in Atlassian-centric shops |

**Recommendation to give in the demo:** if the org is already using Jira, Bitbucket integrates natively; if the priority is a single-platform "VCS + CI + Security" story, GitLab wins; if maximum ecosystem/integration breadth and the largest hiring pool of engineers familiar with the tool matters, GitHub wins. State clearly which one *this* org picked and why (tie back to Sprint-1's conclusion).

---

## 5. Commands

### `git clone`
- **Syntax:** `git clone <url> [directory]`
- **Internally:** opens a connection to the remote, negotiates protocol version, receives a full packfile of all objects, and creates `.git/` locally with a checked-out working tree of the default branch.
- **Common mistake:** cloning with HTTPS then wondering why SSH-configured deploy keys don't work — protocol choice must match the auth method configured.

### `git fetch` vs `git pull`
- `git fetch` downloads new objects/refs from remote but does **not** touch your working directory or current branch.
- `git pull` = `git fetch` + `git merge` (or `git rebase` if `pull.rebase=true`) — this is the #1 source of "surprise merge commit" interview questions.

### `git branch`, `git checkout` / `git switch`
- Creating a branch just creates a new lightweight pointer (a 40-byte SHA reference) — this is why Git branching is described as "cheap," unlike SVN.

### `git merge` vs `git rebase`
- `merge` creates a new commit with two parents, preserving true history.
- `rebase` re-writes commits on top of a new base — cleaner linear history but rewrites SHAs, which is why you **never rebase a shared/public branch**.

### `git cherry-pick`
- Applies the diff introduced by a specific commit onto the current branch as a new commit — used heavily for hotfix backports.

### `git stash`, `git reset`, `git revert`
- `reset` moves the branch pointer (and optionally working tree/index) — history-destructive if pushed.
- `revert` creates a new commit that undoes a previous one — safe for shared branches, preserves history (this is what you use on `main`, never `reset --hard` + force-push).

**Interview trap question:** *"If you accidentally pushed a secret to `main`, what do you do?"* — Answer: rotating the secret immediately is step one (rewriting history doesn't help once it's been fetched by anyone/any CI); then `git filter-repo`/BFG to scrub history and force-push with branch protection temporarily lifted, with the whole team notified to re-clone.

---

## 6. Enterprise Workflow Notes

- Fortune 500 orgs almost never allow direct pushes to `main` — enforced via branch protection rules at the platform level (not gentleman's agreement).
- VCS platform itself is usually backed by the vendor's HA/DR (SaaS) or, for self-hosted, a multi-AZ deployment with the Git object storage on replicated block/object storage and a documented RPO/RTO.
- All access changes go through an audit trail (SOC2/ISO27001 evidence) — this is why manual/ad-hoc permission grants are explicitly called out as an anti-pattern in this ticket's "should go through SSO" requirement.

---

# TICKET 2: Setup Repos (Mono vs Micro)

## Introduction
This ticket decides **how many repositories** the org's codebase is split across, and organizes them (e.g., a `devops` repo for pipeline/Ansible/IaC code, separate from application repos).

## Mono-repo vs Micro-repo — Full Comparison

| Factor | Mono-repo | Micro-repo (Poly-repo) |
|---|---|---|
| Code sharing | Trivial (same repo, same commit) | Requires versioned packages/artifacts |
| Atomic cross-service changes | Single PR/commit | Requires coordinated PRs across repos |
| CI complexity | Needs path-based triggers to avoid rebuilding everything | Simple, each repo has its own pipeline |
| Access control granularity | Coarse (whole repo) unless using CODEOWNERS/path perms | Fine-grained per repo |
| Clone/checkout size | Grows large over time (needs sparse-checkout at scale) | Small, fast |
| Tooling maturity needed | High (Bazel, Nx, or custom path-filtering) at Google/Meta scale | Low — plain Git suffices |
| Best for | Tightly coupled services, shared libraries, small-to-mid teams | Independently deployable microservices, many teams |

**Decision made (following Sprint-1's recommendation):** a **micro-repo** approach — separate repos per application (React frontend, Java backend, Python attendance/notification services) plus a dedicated **DevOps repo** holding Jenkins Shared Libraries, Ansible roles, and IaC. Rationale: independent deploy cadences per service, clear ownership boundaries, and avoiding the tooling investment (Bazel/Nx) that mono-repos demand at scale — not justified for a team of this size yet.

## Internal Working
Git itself has no concept of "mono vs micro" — it's purely an organizational decision layered on top. The only Git-internal factor that matters at scale is repo size and object count, since operations like `git status`/`git gc` degrade with pack size, which is exactly why hyperscale mono-repos (Google) don't use plain Git at all — they use custom VFS-backed systems (Microsoft's VFS for Git / GVFS for Windows OS repo).

---

# TICKET 3 & 4: Authentication & Authorization Setup

## Introduction
**Authentication (authn)** answers "who are you?" **Authorization (authz)** answers "what are you allowed to do?" These are architecturally separate concerns even though they're configured in the same admin panel.

**Why SSO over manual provisioning?**
Manual provisioning means an admin creates a local account per tool per person — this doesn't scale, has no central de-provisioning (a terminated employee retains access until someone remembers to revoke it per tool), and produces no unified audit log. SSO via SAML/OIDC centralizes identity at the IdP so termination = one action revokes everywhere federated.

## Internal Working — SSO Flow (detailed)
1. **SP-initiated flow:** user navigates to `github.company.com` → GitHub Enterprise redirects to IdP's SSO URL with a SAML `AuthnRequest`.
2. IdP authenticates (password + MFA), issues a signed SAML **Response** containing an **Assertion** (NameID = user identifier, Attributes = email, groups).
3. Browser POSTs this assertion back to the VCS's ACS (Assertion Consumer Service) URL.
4. VCS validates the XML signature against the IdP's public cert (established during SAML metadata exchange), checks assertion validity window (`NotBefore`/`NotOnOrAfter` to prevent replay), then creates a session.
5. **Group-to-team mapping** (this is where **authorization** kicks in): the VCS reads the `groups` attribute from the assertion and maps to internal teams with pre-defined repo permissions (Read/Write/Admin) — this is how authz stays declarative/policy-based instead of manual per-user checkboxes.

## Authorization Policy Design (manual policies, since no SSO group mapping is used for fine authz per requirements)
- Define roles: `Reviewer`, `Lead`, `Contributor`, `Admin`.
- Repo-level: `Contributor` = push to feature branches only, no direct push to `main` (enforced by branch protection).
- `Lead` = merge rights on `main` (satisfies PR ticket's "Lead should be able to merge" rule).
- `Reviewer` = read + review/approve rights, no merge rights.
- All of this configured as CODEOWNERS + branch protection rules, not ad-hoc.

---

# TICKET 5: Commit & PR Workflow

## Introduction
This ticket encodes the org's code-review contract as **enforced automation**, not a wiki page nobody reads.

## Acceptance Criteria → How Each Is Actually Implemented

**1. Source branch `feature-XXX`, target `main`**
- Enforced via a **branch naming policy** (GitLab: push rules with regex `^feature-`; GitHub: a status check via a GitHub Action that inspects `github.head_ref` and fails the check if the pattern doesn't match `^feature-[A-Za-z0-9_-]+$`).

**2. Two reviewer sign-offs**
- Branch protection rule: **Required approving reviews = 2**, with "Dismiss stale approvals on new commits" enabled so a late change can't sneak past an earlier approval.

**3. One system (Jenkins) sign-off**
- Jenkins registers as a **required status check** (via the GitHub Checks API or a commit status API call: `POST /repos/{owner}/{repo}/statuses/{sha}`). Internally: Jenkins' GitHub plugin posts a `pending` status when the build starts, then `success`/`failure` on completion. GitHub's merge button stays disabled until this specific context reports `success`.

**4. Code merge validated**
- This is the combination of #2 + #3 — GitHub/GitLab won't enable the "Merge" button until *all* required checks (human + system) are green — this is a platform-level gate, not a social convention.

**5. Lead-only merge**
- Merge button visibility/enablement is additionally restricted via "Restrict who can merge to matching branches" set to the `Leads` team, even after checks pass — this is a *separate* authorization layer on top of the check-passing gate.

## Standards Called Out
- **Commit message standard:** Conventional Commits format (`feat(auth): add SSO login flow (JIRA-123)`), enforced by the pre-commit hook ticket below.
- **PR title/description standard:** template enforced via `.github/pull_request_template.md` requiring: What changed, Why, How tested, Linked JIRA ticket, Screenshots (if UI).
- **Review comment standard:** comments must state *why*, not just *what* ("this could leak PII in logs" not just "change this").

---

# TICKET 6: Notifications (Code Events + Branch Merge)

## Intro
Notifications close the feedback loop — nobody should have to poll the VCS UI to know a PR needs attention or that `main` changed.

## Step-by-Step Setup Guide (Slack + Email)

1. **Create a Slack app / incoming webhook** in the target Slack workspace; scope it to a specific channel (e.g., `#devops-alerts`).
2. **In the VCS platform**, configure an **Integration/Webhook**: GitHub → Settings → Webhooks → Add webhook → Payload URL = Slack's incoming webhook URL (or, more robustly, point to a small Lambda/API Gateway that reformats the payload before forwarding to Slack, since Slack expects its own JSON schema).
3. **Select events** to fire on: `pull_request` (opened, synchronize — i.e., updated, and reviewed/commented), `push` (filtered to `main`/`develop` only via payload inspection in the relay function), `create`/`delete` (branch created/deleted).
4. **Filter for merge-only, stakeholder-only notification** (second requirement): rather than notifying on every push to `main`, filter specifically for the `pull_request` event with `action: closed` AND `merged: true` in the payload — a raw `push` event doesn't semantically distinguish "someone merged a PR" from "someone pushed directly," so a relay function must inspect the merge flag.
5. **Restrict merge rights** (only reviewers/leads can merge) is enforced at the branch protection layer (see Ticket 5) — notification logic doesn't grant this, it only reports it.
6. **Route to specific stakeholders**: use a dedicated Slack channel with only the required members, and for email, a distribution list (e.g., `leads-devops@company.com`) rather than broadcasting to `all-engineering`.
7. **Test:** open a dummy PR, merge it, confirm exactly one Slack message + one email fire, confirm non-target branches produce no notification.

## Conclusion
This setup ensures visibility without notification fatigue — a common enterprise anti-pattern is over-notifying, which trains people to ignore the channel entirely.

## Contact Information
DevOps Team Lead — devops-lead@company.com | Slack: #devops-support

## References
- GitHub Webhooks documentation
- Slack Incoming Webhooks documentation

---

# TICKET 7: Pre-commit Hook (JIRA Commit Message Mapping)

## Introduction
A **pre-commit hook** is a script Git runs locally before a commit object is finalized — if the script exits non-zero, the commit is aborted. This ticket enforces that every commit message references a valid JIRA ticket ID, catching non-compliant commits **before** they ever leave the developer's machine (shifting the check left, versus catching it later in CI).

## Step-by-Step Setup Guide

1. Install the `pre-commit` framework (Python-based, works language-agnostically): `pip install pre-commit`.
2. Create `.pre-commit-config.yaml` in repo root defining a local hook that runs a regex check against `commit-msg` (note: JIRA-ID validation must live in the **commit-msg** hook stage, not `pre-commit` stage, since the commit message doesn't exist yet at the `pre-commit` stage).
3. Hook script (conceptually): read `$1` (path to the commit message file Git passes in), apply regex `^[A-Z]{2,10}-\d+:` (e.g., `JIRA-123: fix login bug`), exit 1 with a helpful error if no match.
4. Install the hook locally: `pre-commit install --hook-type commit-msg`.
5. Demo: attempt a commit without a JIRA ID → hook rejects it with a clear error message; attempt with a valid ID → commit succeeds.
6. **Enterprise note:** local hooks are opt-in per clone (a dev could skip installing it, or use `--no-verify`), so mature orgs *also* re-validate the same regex as a required CI/server-side check (GitHub Actions or a server-side `commit-msg` hook via GitLab's push rules) — local hook = fast feedback, server-side check = the actual enforcement boundary.

## Conclusion
Pre-commit hooks shift quality checks left, reducing CI failures and keeping JIRA traceability intact for every change — critical for audit and release-note automation.

## Contact Information
DevOps Team Lead — devops-lead@company.com

## References
- pre-commit.com framework docs
- Atlassian JIRA smart-commit documentation

---

# Reviewer & Interview Questions — VCS Epic

**Reviewer questions (sample of the most likely 20):**
1. Why did we choose [chosen platform] over the alternatives?
2. Walk me through what happens end-to-end when a PR is merged.
3. How do we prevent a direct push to `main`?
4. How is Jenkins' sign-off technically enforced, not just a suggestion?
5. What happens if someone disables their pre-commit hook — are we still protected?
6. How does SSO de-provisioning work when someone leaves the company?
7. Why micro-repo instead of mono-repo for this project?
8. What's the difference between a webhook payload for `push` vs `pull_request` events, and why does that matter for our merge-notification logic?
9. How would you roll back if a bad commit reaches `main`?
10. What's stored in `.git/objects` and why does Git use content-addressable storage?

**Interviewer-style / cross-questions:**
1. "If GitHub is down, how does your team keep working?" → local Git still works fully offline for commits/branches; only push/pull/PR/CI is blocked — good opportunity to discuss DR.
2. "Your Jenkins check is green but a Lead says the code is broken — should the merge button still be enabled?" → discuss limits of automated checks vs human override, and why Lead-restricted merge exists as a safety net.
3. "How would this design change at 10x the current team size?" → discuss mono-repo tooling investment, CODEOWNERS scaling, sharded pipelines.

**Production incident scenarios:**
1. A secret was committed and pushed to `main` before anyone noticed — walk through response.
2. The webhook to Jenkins stopped firing silently for two days — how do you detect and debug this (check delivery logs in VCS's webhook settings, verify Jenkins endpoint reachability/firewall, check for payload signature mismatches).
3. Two required reviewers approved, but a third person's follow-up comment ("wait, this breaks X") was never addressed because approvals aren't dismissed on new pushback comments — discuss why "dismiss stale reviews on new commits" matters, and where comment-only feedback fits outside the approval gate.

---

*End of Part 1. Next up: Part 2 — Application CI Design (Cred Scanning, License Scanning, AMI, Commit Sign-off, and the full React/Java/Python/Go CI check matrix). Let me know if you want that next, or want to reorder the series.*
