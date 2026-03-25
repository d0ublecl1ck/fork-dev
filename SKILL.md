---
name: fork-dev
description: Create and maintain a private development mirror of a public upstream repository without using the hosting platform's Fork button. Use when a user needs to secondary-develop an open-source project privately, keep proprietary changes out of a public fork, set up origin/upstream remotes, sync upstream changes, or resolve conflicts during upstream merges and rebases on GitHub, GitLab, or similar platforms.
---

# Fork Dev

Set up a private mirror workflow when the user needs both of these outcomes:
- Keep the working repository private.
- Continue pulling updates from a public upstream project.

Do not use the platform Fork button for this workflow. Create a new empty private repository and mirror the public upstream into it manually.

Collect the missing private repository name before running commands. Default the rest of the workflow to the house convention unless the repository state proves otherwise.

## Decide the path

Use this skill for either of these cases:
- Create a new private derivative of a public upstream repository.
- Repair an existing workflow where the user forked publicly first and now needs a private mirror with a clean `origin` and `upstream` setup.

If the user only wants a normal public fork and does not care about private derivative work, this skill is unnecessary.

## Gather the only required user input

Ask only this when it is missing:
1. What is the private repository name?

Do not ask for platform, owner, transport, or branch names by default.

Use these defaults unless the repository state or user instruction says otherwise:
- Hosting platform: GitHub.
- Owner or namespace: the currently authenticated `gh` account or its target org chosen by the user in the active workflow.
- Remote transport: use the authenticated `gh` default and then read back the actual URL from `gh repo view`.
- Upstream integration branch: `sync-upstream`.
- Internal development branch: `main`.

If the existing repository layout contradicts these defaults, inspect the real state and adapt instead of asking speculative questions.

## Prefer GitHub CLI when GitHub is the target

If `gh` is available, prefer GitHub for repository creation and remote validation.

Check first:

```bash
gh auth status
```

If authentication is missing or invalid, instruct the user to authenticate before continuing.

Resolve the owner automatically first:

```bash
gh api user --jq .login
```

Create the empty private repository with the resolved owner and confirmed name:

```bash
gh repo create <owner>/<repo-name> --private --disable-issues --disable-wiki --confirm
```

Then capture the resulting remote URL that matches the user's preferred transport:

```bash
gh repo view <owner>/<repo-name> --json sshUrl,url
```

Use the returned URL that matches the local authenticated setup. Do not ask the user to choose SSH versus HTTPS unless there is an actual auth problem that requires switching transport.

If the repository already exists, verify it instead of recreating it:

```bash
gh repo view <owner>/<repo-name> --json nameWithOwner,isPrivate,sshUrl,url,defaultBranchRef
```

Reject the workflow if the target repository is not private.

## Standard remote model

Aim for this final state:
- `origin` points to the user's private repository.
- `upstream` points to the official public repository.

Verify with:

```bash
git remote -v
```

If the remote layout is wrong, fix it before any sync work.

## Initialize a private mirror

Use this sequence when starting from scratch:

1. Create a new empty private repository on GitHub, GitLab, or the target host.
2. Mirror the public upstream into that private repository.
3. Re-clone the private repository as the normal working copy.
4. Add the public repository back as `upstream`.

Commands:

```bash
git clone --bare <upstream-url>
cd <repo-name>.git
git push --mirror <private-origin-url>
cd ..
rm -rf <repo-name>.git
git clone <private-origin-url>
cd <working-copy>
git remote add upstream <upstream-url>
git remote -v
```

Rules:
- Require the private destination repository to be empty before `git push --mirror`.
- Prefer `git clone --bare` plus `git push --mirror` so branches, tags, and history stay intact.
- Re-clone after mirroring instead of developing inside the bare repository.
- If the host is GitHub, prefer creating the destination repository with `gh repo create` instead of asking the user to click through the UI.

## Convert an existing public fork workflow

If the user already clicked Fork on the hosting platform:

1. Keep the local repository if it already contains useful work.
2. Create a brand-new empty private repository.
3. Repoint `origin` to the new private repository.
4. Ensure `upstream` still points to the official public repository.
5. Push all needed branches and tags to the new private repository.

Typical commands:

```bash
git remote rename origin old-origin
git remote add origin <private-origin-url>
git remote add upstream <upstream-url>   # only if missing
git push -u origin <main-branch>
git push origin --tags
```

If multiple long-lived branches matter, push them explicitly or mirror from a clean clone.

## Sync upstream changes safely

When the private mirror already exists, prefer this baseline flow:

```bash
git fetch upstream
git checkout sync-upstream
git merge upstream/<upstream-default-branch>
git push origin sync-upstream
git checkout main
git merge sync-upstream
git push origin main
```

Use `sync-upstream` as the upstream integration branch and `main` as the internal development branch by default.

Before syncing:
- Check for uncommitted changes with `git status --short`.
- Confirm the current branch with `git branch --show-current`.
- Fetch first; do not merge blind.
- Verify the branch layout if the repository state suggests a different convention.

## Resolve conflicts pragmatically

When merge or rebase conflicts appear:
- Inspect conflicted files with `git status`.
- Resolve files one by one instead of bulk accepting changes.
- Prefer preserving the user's proprietary business logic while absorbing upstream structural or security fixes.
- After resolving a merge, run `git add <files>` and `git commit`.
- After resolving a rebase, run `git add <files>` and `git rebase --continue`.

If the conflict is broad:
- Identify whether the user customized the same abstraction boundary that upstream changed.
- Consider extracting the customization behind a thinner adapter or extension layer before continuing.

## Verification checklist

Before declaring the workflow complete, verify all of these:
- `origin` is private and writable by the user.
- `upstream` is the official public source.
- `git remote -v` shows the expected mapping.
- `git branch -a` includes the expected local and remote branches.
- `git fetch upstream` succeeds.
- The target integration branch is in sync after merge or rebase.

## Persist the workflow into AGENTS.md

After the private mirror workflow is established, automatically write the repository-specific sync and development guidance into the target project's `AGENTS.md`.

Write or update these facts with the real resolved values from the completed setup:
- `origin` remote URL.
- `upstream` remote URL.
- The primary local working copy path.
- The internal development branch.
- The upstream integration branch.
- The standard sync command sequence for this repository.
- Any repository-specific warning about long-lived divergence or merge precautions that was discovered during setup.

Rules:
- Update the target project's `AGENTS.md`, not a global rule file, unless the user explicitly asks for a system-wide rule.
- If `AGENTS.md` does not exist in the target repository, create it.
- Prefer updating an existing matching section instead of duplicating sync instructions.
- Keep the recorded guidance concise, operational, and repository-specific.
- Do not finish the task immediately after remote setup; include the `AGENTS.md` update as part of completion.

Use this fixed section title and field order so the output stays consistent across repositories:

```md
## Git Mirror Workflow

- Private mirror remote is `origin`: `<origin-url>`
- Official upstream remote is `upstream`: `<upstream-url>`
- Primary local working copy path is `<local-worktree-path>`
- Internal development branch is `<internal-dev-branch>`
- Upstream integration branch is `<upstream-integration-branch>`
- Do not merge `<upstream-remote>/<upstream-default-branch>` directly into `<internal-dev-branch>` without first updating the local `<upstream-integration-branch>` branch and reviewing the delta<optional-warning-suffix>
- Use this standard sync flow unless the user explicitly requests a different branch strategy:

```bash
cd <local-worktree-path>
git fetch upstream
git checkout <upstream-integration-branch>
git merge upstream/<upstream-default-branch>
git push origin <upstream-integration-branch>
git checkout <internal-dev-branch>
git merge <upstream-integration-branch>
git push origin <internal-dev-branch>
```
```

Template rules:
- Keep the section title exactly `## Git Mirror Workflow`.
- Keep the bullet order exactly as shown.
- Replace placeholders with resolved real values.
- Omit `<optional-warning-suffix>` when no special warning is needed.
- If the repository already has a `## Git Mirror Workflow` section, update it in place instead of appending a second copy.

## Communication pattern

When helping the user, be explicit about:
- Which repository should be private.
- Which remote is `origin`.
- Which remote is `upstream`.
- That `sync-upstream` is the branch used to receive upstream merges by default.
- That `main` is the internal development branch by default.

Use concrete command sequences with the actual branch names when they are known. Default to `sync-upstream` and `main` unless the repository already uses a different layout.

When the user has not yet provided the private repository name, ask only for that value instead of expanding into a multi-question intake.
