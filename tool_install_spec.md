# Tool Install Spec — Secure Installation of Tools via Telegram Commands

Date: 2026-01-30
Scope: Design a secure, auditable mechanism that allows a user to install tools (code, workflows, deploy keys, actions) through Telegram commands addressed to an assistant, while safely granting the assistant the minimum required permissions.

Goals
- Minimal manual steps for the user.
- Strong authentication and verification linking Telegram identity to remote provider identity (GitHub/GitLab/etc.).
- Short-lived, least-privilege credentials issued to the assistant where possible.
- Auditable, tamper-evident logs of every step.
- Defenses against CSRF, replay, brute-force, and social-engineering attacks.

High-level command flow
1. User issues a Telegram command to the assistant, e.g.:
   - /install_tool github.com/org/repo@ref
   - /install_tool repo:my-org/my-repo branch=main method=deploy-key
2. Assistant validates basic formatting and responds with a short acknowledgement and an ephemeral action URL (HTTPS) and a one-time confirm code shown in Telegram.
3. User opens the URL (or uses in-app Telegram login button) to a small web UI under your control. The web UI requires explicit, interactive confirmation and an authentication step that proves control of the target installation account (GitHub/GitLab) and links that account to the requesting Telegram user.
4. After successful verification, the web UI presents a minimal set of choices and requested scopes/permissions. The user selects the method (one-time PAT exchange, GitHub Action, SSH deploy key) and consents.
5. Backend performs the requested install using the chosen, least-privilege approach, issuing short-lived credentials to the assistant where possible; stores the resulting credentials encrypted; returns status to user and logs the event.
6. Assistant posts results in Telegram (or the web UI shows it); tokens/keys are automatically expired/revoked after the operation or at an agreed lifetime.

Design details

1) Authentication & Identity Binding
- Telegram identity capture: every command includes the Telegram user id and message id. The assistant generates a random request id (UUIDv4) and a 6–8 character human-friendly confirmation code shown in the Telegram reply.
- Web UI authentication options (choose at least one):
  - OAuth (recommended): Use the provider's OAuth (GitHub/GitLab) to authenticate the user. OAuth provides consent screens and account proofing.
  - Telegram login widget: optionally allow the Telegram login widget to assert Telegram identity for correlation.
  - OAuth + Telegram confirmation: require both provider OAuth and the one-time Telegram confirmation code to complete the mapping of Telegram -> provider account.
- Binding: store mapping request_id -> telegram_user_id -> provider_account_id in the request record.

2) Verification & Authorization
- Verify repository ownership or admin role before any write operation.
  - Use provider APIs to check if the authenticated account has admin/maintainer rights to the target repo/org.
  - If not an admin, require an explicit invitation step (see 'Admin approval') or refuse.
- Admin approval options:
  - Single admin: built-in flow presents an approval request to repo admins (email/DM/PR comment) with the request id and link.
  - Organization policy: support RBAC groups — only group members can approve.

3) CSRF / CAPTCHA / Rate-limiting
- The web UI must use standard anti-CSRF (synchronizer tokens) for forms and state-changing endpoints.
- Rate-limit requests per Telegram user, per IP, and per target repo. Add exponential backoff.
- If suspicious patterns occur (multiple requests, rapid retries, IP mismatch between Telegram request and web UI), require CAPTCHA (reCAPTCHA or hCaptcha) and additional 2FA (e.g., TOTP) before proceeding.

4) One-time Request Confirmation
- The assistant's Telegram reply must include the one-time confirmation code and a short expiration window (e.g., 10 minutes).
- The web UI requires both that code and OAuth proof to proceed. This prevents attackers who intercept a link from finalizing without access to the Telegram account.

5) Credential Handling and Token Storage
Principles:
- Never store raw long-lived user tokens/PATs if avoidable.
- Prefer provider-native short-lived tokens (OAuth tokens with short refresh, GitHub App installation tokens, ephemeral deploy keys) or create ephemeral credentials that the assistant controls for a short time.

Storage model:
- Secrets are encrypted at rest with a KMS (Cloud KMS or an HSM). Keys for KMS are managed by the operator and rotated regularly.
- Store only metadata and a reference (key ID or secret id) in the database. Do not log raw secrets.
- Use envelope encryption: service encrypts secrets with a data key; data key encrypted with KMS.

Least-privilege suggestions by method:
- OAuth/GitHub App: register a GitHub App that requests only necessary permissions (contents: read/write only for the single repo or issues for PR integration). Use installation tokens that are short-lived (1 hour). Use App to perform actions via API on behalf of the app installation.
- One-time PAT exchange: accept PAT only temporarily, use it to create a short-lived installation token or to add a deploy key/secret on the repo, then immediately revoke or delete the PAT. If possible avoid storing PAT at all.
- SSH deploy key: generate an ephemeral key pair on the server and ask the user to add the public key as a deploy key with an expiration timestamp. Alternatively, perform API call to add the key while the user is OAuth-authenticated (so we don't ask for private key transfer).

Rotation & expiration policies:
- Tokens created by the system should have max TTL = 1 hour by default. For low-privileged actions, TTL = 5–15 minutes.
- If provider allows key expiration (e.g., expiring deploy keys), set expiration at creation. If not, implement an automated revocation job that removes keys after TTL.

6) Audit logging
- All important events are logged: request creation, Telegram user id, request id, OAuth account id, IPs seen, token issuance, token revocation, API calls made, files changed, and confirmations.
- Logs are append-only and HMAC-signed daily with a rotated sign key to provide tamper evidence. Store logs in a secure, write-once store where possible (e.g., append-only S3 bucket with object lock, or database with immutable audit table and restricted write privileges).
- Each log entry should contain: timestamp, request_id, telegram_user_id, provider_user_id, request_payload, action_result, operator_id (if any), and hash chaining to previous log entry (or HMAC).

7) Interaction & Transparency
- Always show a summary of actions before executing (e.g., "I will add a deploy key to repo X with write access until YYYY-MM-DD. Proceed?"). The user must confirm in the web UI.
- Provide a link to the audit record for that request and the exact API operations performed (with timestamps and minimal identifiers—not secrets).

Sample Implementation Options

Option 1 — One-time short-lived PAT entry flow (user-driven minimal friction)
Overview: The user pastes a one-time PAT into the secure web UI that is ephemeral and used only to perform a limited set of operations (create repo secret, add deploy key, or create GitHub App installation). The PAT is validated and immediately deleted after use.

Flow:
1. Telegram: /install_tool repo:org/name method=pat
2. Assistant: responds with request_id, one-time code, and secure HTTPS URL
3. User opens URL, OAuth optional, then pastes PAT into a short-lived form
4. Server validates PAT scopes by calling the provider's API (e.g., GET /user and test create-key endpoints or check scopes header), confirms admin rights, and shows the exact operations to be performed.
5. On confirmation, server uses PAT to perform actions (create repo secret, add deploy key, create a GitHub App Installation, add workflow file). Server immediately deletes PAT from memory, and if possible revokes PAT via provider API or instructs user how to revoke manually.
6. Server stores only resulting ephemeral tokens/keys with TTL and logs the operations.

Safety checks & script snippet (Node/Express, conceptual):
- Validate scopes header returned by the provider.
- Confirm user is repo admin (GET /repos/:owner/:repo/collaborators/:username/permission).
- Use a forced command wrapper for any SSH actions.

Pseudo-code (high level):
- verifyPAT(pat) -> GET /user, GET /repos/:owner/:repo -> ensure admin
- performInstall(pat) -> create-deploy-key or create-secret
- deletePATFromMemory()

Security notes:
- Prefer showing a script the user can run locally instead of uploading PAT to the service if they are unwilling to enter the PAT.
- Use short expiry and auto-revoke.

Option 2 — Deployable GitHub Action pattern (recommended for reproducibility & auditability)
Overview: Use GitHub Actions with manual approval (workflow_dispatch) or a GitHub App + workflow to run the install steps in the repo context. This avoids transferring long-lived credentials to your service and leverages GitHub's runner environment, audit log, and repository-level secrets.

Flow A — PR + workflow_dispatch (safer when you need human review):
1. Assistant creates a PR that adds an `/.github/workflows/install-tool.yml` workflow and an installation script in the repo.
2. PR is opened and visible to repo admins. Admin reviews the PR and merges it when satisfied. Optionally, the assistant posts instructions to trigger workflow_dispatch to run the installation.
3. Workflow runs in GitHub runner context with the repo's GITHUB_TOKEN (limited privileges) and performs the install tasks.
4. Workflow includes explicit logging to the repo Actions logs (auditable) and posts a completion comment to the PR.

Flow B — GitHub App + repository dispatch (automated with consent):
1. User authenticates via OAuth and grants permission to the App for the specific repo.
2. Assistant triggers a repository_dispatch event to start a pre-existing workflow in the repo that the user prepared (or the assistant added via PR earlier).
3. The workflow runs with GITHUB_TOKEN or a repo secret, performs the install, logs results.

Example workflow (install-tool.yml):
- on: workflow_dispatch
  jobs:
    install:
      runs-on: ubuntu-latest
      steps:
      - uses: actions/checkout@v4
      - name: Run install script
        run: ./scripts/assistant_install.sh

Security & audit benefits:
- Actions run in provider's environment, so the assistant never directly needs repo write credentials.
- Logs live in the Actions logs (auditable). PR flow ensures manual review.

Option 3 — SSH deploy key method (for systems that require SSH)
Overview: The server generates an ephemeral SSH key pair and requests the user (via OAuth) to add the public key as a deploy key with a specified expiry and limited access. Alternatively, the server (with OAuth) may add the key via provider API.

Flow:
1. Assistant generates key pair and shows public key + instructions to add as a deploy key with expiration and limited permissions.
2. User adds the public key via provider UI or allows the assistant to add the key via API while authenticated.
3. Assistant connects via SSH to run the installer with a forced command or limited shell.
4. On completion, key is removed.

Safety checks:
- Generate key with strong algorithm (ed25519) and use a meaningful comment containing request_id and expiry.
- Use forced-command in authorized_keys if possible to restrict remote actions to a single script that verifies the request id and logs actions.
- Ensure the server verifies host fingerprints (avoid MITM).
- Revoke or remove the key immediately after use; if the provider supports expiring keys, set expiration at creation.

Sample SSH generation snippet (conceptual):
- ssh-keygen -t ed25519 -f /tmp/request-<id> -N "" -C "tool-install request:<id> expires:<iso>"
- present public key to user or add via API

Risk assessment

Primary risks:
- Credential compromise (user uploads long-lived PAT to the assistant service).
- Account takeover via link interception or social engineering.
- Excessive permissions granted to assistant leading to unwanted code changes.
- Replay or CSRF attacks when confirmation links are abused.
- Tampering or deletion of audit logs.

Mitigations (summary):
- Prefer OAuth / GitHub App over PAT; prefer short-lived tokens and ephemeral keys.
- Always require proof of both Telegram presence (confirmation code) and provider authentication (OAuth).
- Use CSRF tokens and short expirations on confirmation URLs.
- Rate-limit, CAPTCHA, and require 2FA for high-risk requests.
- Store secrets encrypted with KMS; restrict access with IAM roles.
- Append-only, HMAC-signed audit logs with restricted write ACLs.
- Default deny: require explicit admin consent for repo writes. Use PR + Actions or explicit OAuth grant.

Recommended default policy (conservative)
- Deny by default. Users must opt in per-repo or per-org.
- Required steps before any install:
  1. Telegram command issued by a mapped user.
  2. User completes OAuth to provider and is verified as repo admin.
  3. User confirms action on the web UI using the one-time confirmation code (10 minute expiry).
  4. Use the GitHub Action workflow/PR pattern by default. Only allow automatic deploy-key or PAT flows if the organization explicitly enables them.
- Token lifetime: 15 minutes for ephemeral tokens used for installation; 1 hour max for app installation tokens.
- Logs retention: at least 90 days in hot storage; 365 days archived (policy configurable).

Operational checklist for implementers
- Build a minimal web UI that: accepts request_id + displays one-time code; enforces CSRF tokens; supports OAuth for GitHub/GitLab; requires user confirmation on exact operations.
- Implement a secrets manager + KMS envelope encryption.
- Register a GitHub App with minimal permissions for automation where possible.
- Implement HMAC-signed append-only audit logs.
- Rate-limit, CAPTCHA, IP checks.
- Add monitoring & alerting for suspicious patterns (many installs from same user/ip, repeated failed attempts).

Appendices: short example scripts

1) PAT verification (Node pseudocode):

async function verifyPat(pat, repo) {
  const headers = { Authorization: `token ${pat}`, Accept: 'application/vnd.github+json' };
  const user = await fetch('https://api.github.com/user', { headers }).then(r=>r.json());
  // Check repo admin permission
  const perm = await fetch(`https://api.github.com/repos/${repo}/collaborators/${user.login}/permission`, { headers }).then(r=>r.json());
  if (!['admin','maintain'].includes(perm.permission)) throw new Error('not admin');
  // Check scopes via response header (or do a test write guarded)
  return { login: user.login, id: user.id };
}

2) Add ephemeral deploy key (conceptual):
POST /repos/:owner/:repo/keys
body: { title: `tool-install <request_id> expires=<iso>`, key: "ssh-ed25519 AAAA...", read_only: false }

3) GitHub Actions workflow (install-tool.yml):
on: [workflow_dispatch]
jobs:
  install:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Run assistant installer
      run: |
        set -euo pipefail
        ./scripts/assistant_install.sh "$INPUT_REQUEST_ID"

Closing notes
- Prefer GitHub App + Action workflow or PR flow for the safest, most auditable option.
- If PATs or deploy keys are required, make them ephemeral and delete/revoke them immediately after use.
- Keep human confirmation and provide clear, auditable records of every action.

---
Saved by subagent: tool-install-spec

