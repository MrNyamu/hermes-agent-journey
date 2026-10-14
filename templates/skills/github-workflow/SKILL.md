---
name: github-workflow
description: How to clone, branch, commit, push and open pull requests on the user's GitHub repos from the sandbox, with per-owner tokens. Use for any task that touches a GitHub repository.
version: 1.0.0
metadata:
  hermes:
    tags: [github, git, pull-requests, engineering]
    category: engineering
---

# GitHub workflow

## When to use
Any task that reads or changes code in a GitHub repository, or touches PRs, issues or CI.

## The setup (already done, don't change it)
- Tokens live in `~/.tokens/<owner>`, one per GitHub owner: e.g. `my-user`, `my-org`. Never print, copy, cat or echo them, and never put them in files, commits, logs or chat.
- **git** picks the right token automatically from the repo URL (credential helper `/root/bin/git-credential-ghtok`). Use plain `git`.
- **gh**: always call `/root/bin/gh`, never a bare `gh`. It picks the token from the current repo's remote.
  - Outside a repo, set the owner: `GH_OWNER=my-org /root/bin/gh repo list my-org`
  - Or pass `-R owner/repo`.
- Do not run `gh auth login`, `gh auth token`, or change git credential config.
- If `ghtok` says "no token for <owner>", stop and tell the user. That owner isn't set up yet.

## Procedure
1. **Workspace.** Work under `/workspace/<repo>`. If the repo is already cloned, `git fetch` and update main. Otherwise clone it with `git clone https://github.com/<owner>/<repo> /workspace/<repo>`.
2. **Read first.** Before editing, read the repo's `AGENTS.md`, `CLAUDE.md` or `README.md`, and follow their conventions.
3. **Branch.** Create a new branch off the latest default branch, named `<type>/<short-description>`, where the type is `feat`, `fix`, `docs`, `chore` or `refactor`.
4. **Change.** Make focused changes. Run the project's tests and linters if they exist and fit on this box (8 GB RAM, so no Flutter or mobile builds; leave those to CI).
5. **Commit.** Write clear messages in the repo's existing style. Make small, logical commits.
6. **Push.** Run `git push -u origin <branch>`.
7. **Open a PR.** Run `/root/bin/gh pr create --base <default-branch> --title "..." --body "..."`. The body should say what changed, why, how it was tested, and anything left undone or uncertain.
8. **Merge style.** PRs are merged with **rebase and merge**, so every commit lands on main individually: keep commits focused, with clear messages.
9. **Report.** Send the user the PR link and a short summary, then stop.

## Never
- Push to `main` or `master`, force-push to shared branches, or rewrite published history.
- Merge PRs, approve your own PRs, delete branches you didn't create, or change repo settings.
- Edit CI/CD workflows (`.github/workflows/`) or deployment config unless the user explicitly asks.
- Commit secrets, `.env` files or credentials. Check `git diff --staged` before every commit.
- Invent identities, emails, user IDs or credentials. GitHub noreply emails are `<id>+<login>@users.noreply.github.com`, where the ID is a real account ID. Look it up (`/root/bin/gh api user --jq .id`) or ask. A wrong ID credits a stranger's account.
- Change your own git identity (`user.name` / `user.email`) unless the user asks.
- Mix work across owners. Keep each organization's code inside its own repos; never copy code between, for example, an employer's org and your own.

## Pitfalls
- A push rejected with "protected branch" or "pull request required" means you pushed to `main`. Push a branch instead.
- HTTP 403 from the sandbox usually means the host isn't on the egress allowlist. Tell the user which host. Don't try to work around it.
- A 403 or 404 from the GitHub API means the token doesn't cover that repo or permission. Report it and don't retry with other tokens.
- Files in `/workspace` are owned by root. That's expected.

## Verification
Before reporting done, confirm that the branch exists on GitHub, the PR is open against the default branch, and `git status` is clean.
