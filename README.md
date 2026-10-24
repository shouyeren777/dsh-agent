# dsh on GitHub Actions

An agent that lives in a GitHub repository: comment `/dsh <task>` on an issue and a
DeepSeek Harness run picks it up in a throwaway runner, does the work, and replies
in the thread.

```
issue comment "/dsh 修一下这个 bug"
        │  only OWNER / MEMBER / COLLABORATOR may trigger
        ▼
ubuntu-latest (ephemeral, destroyed after the run)
  ├─ npm i -g @deepseek-ai/dsh@0.2.1-rc.3
  ├─ DEEPSEEK_API_KEY  (repo secret)
  ├─ GITHUB_TOKEN      (contents/issues/pull-requests write, nothing else)
  └─ dsh headless "<prompt built from the issue>"
        ▲
        └─ branch + PR (never main), final message posted back as a comment
```

## Files

| Path | Role |
| --- | --- |
| `.github/workflows/dsh.yml` | triggers, permissions, and the reply step |
| `scripts/run-dsh.sh` | builds the prompt, runs `dsh headless`, captures the answer |

## Setup

### 1. Get a DeepSeek API key

`dsh headless` needs `DEEPSEEK_API_KEY`; without it the CLI exits with
`MISSING_CREDENTIAL`. A plain account login is *not* enough in CI — it must be an
API key, billed per token.

### 2. Create the repository and push these files

```bash
git init -b main
git add .
git commit -m "add dsh on actions"
git remote add origin git@github.com:<you>/<repo>.git
git push -u origin main
```

### 3. Add the secret

Repository → **Settings → Secrets and variables → Actions → New repository secret**

- Name: `DEEPSEEK_API_KEY`
- Value: your `sk-...` key

`GITHUB_TOKEN` is supplied by the runner automatically; never create a PAT for this.

### 4. Protect `main`

Settings → Branches → Add branch protection rule for `main`:

- Require a pull request before merging
- Do not allow force pushes
- Do not allow deletions

This is the backstop: even if the agent is talked into something by malicious issue
text, it cannot rewrite the default branch.

### 5. Try it

Open an issue, then comment `/dsh reply with just the word OK`.

The run appears under the **Actions** tab; the answer is posted back to the issue.

## Guardrails, and why each one exists

| Guard | Threat it blocks |
| --- | --- |
| `author_association` must be OWNER/MEMBER/COLLABORATOR | a stranger triggering runs that burn your API quota or push code with your token |
| `startsWith(body, '/dsh')` | the agent reacting to ordinary conversation |
| event fields passed via `env:`, never interpolated into `run:` | shell injection through the comment body |
| issue text wrapped in `--- UNTRUSTED ---` markers + prompt rule | prompt injection: issue text telling the agent to exfiltrate secrets |
| `permissions:` block | a compromised run touching releases, packages, or webhooks |
| ephemeral GitHub-hosted runner | anything the agent does to the machine dying with the run |
| branch protection on `main` | an unreviewed push reaching the default branch |

## Known unknowns to verify on the first run

These are not yet proven in CI:

1. **dsh bootstraps its own profile in a clean `$HOME`.** Locally the CLI reads
   `~/.dsh/profiles`. In CI that directory does not exist; if `dsh` does not seed it
   from its bundle templates, set `DSH_HOME` to a directory the workflow creates.
2. **Sandbox support in a container.** `dsh` may pick its bubblewrap/seccomp bash
   sandbox on Linux; if that fails inside the runner, switch the profile to the
   plain local bash tool.
3. **Wall-clock cost per run.** 25 minutes is the timeout; typical small tasks
   should finish well inside it.

## Do not

- Do not add third-party "dsh action" repositories from random GitHub accounts. An
  action runs with your token; an unverified one is a supply-chain attack waiting
  to happen.
- Do not widen `permissions:` to `write-all`.
- Do not commit `~/.dsh/.credentials.yaml` or any `sk-...` key.
