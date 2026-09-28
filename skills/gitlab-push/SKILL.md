---
name: gitlab-push
description: >
  Use this skill for essentially every commit, push, branch publication,
  or merge request operation involving a GitLab repository. This skill
  applies whenever the user asks to commit changes, push changes, create
  or update a branch, or create a GitLab merge request. GitLab repositories are
  located in folders named like ~/develop/savi*.

  By default, push feature branches to the user's fork, not the upstream
  repository. Feature branches must follow the format
  PRJ-####/PRJ-###-<kebab-case-description>.

  Before creating or switching to a feature branch, inspect Git remotes,
  branches, and recent history to determine the appropriate base branch.
  The base branch will typically be a release branch matching
  release/v#.#.# or release/PRJ/v#.#.#. Do not assume main, master, or
  develop is the base branch.

  When creating a GitLab merge request, use the user's fork as the source
  repository and the canonical/upstream repository as the target repository.
  Target the detected release/base branch unless the user explicitly
  specifies a different target.
---

# Push Commit and Create Merge Request

## Purpose

Use this skill when working in the folders like ~/develop/savi* whenever the user asks to commit, push, create a merge request, open a PR/MR, or otherwise publish local changes

This skill is intended to be invoked for essentially every push from this repository.

The default behavior is:

1. Determine the appropriate base/release branch.
2. Create or use a branch following the repository's required naming convention.
3. Review the changes.
4. Commit the changes with an appropriate commit message.
5. Push the branch to the user's fork.
6. Create a merge request from the fork branch into the appropriate upstream/base branch.
7. Report the resulting branch, commit, and merge request.

Do not push directly to the upstream repository's protected/release branch unless the user explicitly requests it.

---

## Repository and Remote Rules

### Default remote

Use the user's **fork** as the push destination.

Before pushing, inspect the configured Git remotes:

```bash
git remote -v
```

Identify:

* The user's fork, normally the `origin` remote.
* The upstream/canonical repository, if configured as `upstream`.

If the remotes are ambiguous, inspect them before making assumptions.

The normal arrangement is:

```text
origin    -> user's fork
upstream  -> canonical repository
```

Push branches to `origin`.

The merge request should target the canonical repository's appropriate base branch.

If the user explicitly specifies a different remote, repository, fork, or destination, follow the user's instruction instead.

---

## Base Branch Detection

Do not assume the base branch is `main`, `master`, or `develop`.

Before creating the feature branch, inspect the repository's existing branches and recent history:

```bash
git branch -a
git remote -v
git log --oneline --decorate -20
```

Look for release branches matching patterns such as:

```text
release/v#.#.#
release/dsp/v#.#.#
```

Examples:

```text
release/v2.5.6
release/v3.5.6
release/eed/v1.0.1
release/dsp/v1.1.3
```

Prefer the release branch that the current work belongs to.

Use repository context to determine this, including:

* The currently checked-out branch.
* Existing local branches.
* Remote branches.
* Recent commits.
* The ticket/project identifier.
* Existing merge requests, when available.
* Repository documentation/configuration.
* The release branch from which the current work should logically originate.

If there is an obvious current release branch, use it.

If multiple release branches are plausible and the correct one cannot be determined confidently, ask the user before creating the branch or merge request.

Do not silently choose an arbitrary release branch.

Once the base branch is determined, update it before branching:

```bash
git fetch upstream
git checkout <base-branch>
git pull --ff-only upstream <base-branch>
```

If the repository uses a different upstream remote, substitute the appropriate remote.

---

## Feature Branch Naming

Branches must follow this format:

```text
PRJ-####/PRJ-###-<kebab-case-description>
```

Where:

* `PRJ-####` is the parent/project ticket identifier.
* `PRJ-###` is the specific work-item identifier.
* `<kebab-case-description>` is a concise description using lowercase `kebab-case`.

Example:

```text
PRJ-1234/PRJ-567-fix-audio-subscription-cleanup
```

Another example:

```text
PRJ-4321/PRJ-890-add-device-reconnection-handling
```

### Ticket identification

Determine the ticket identifiers from:

1. The user's request.
2. The current branch, if the work is already on a correctly named branch.
3. Existing commits or issue references.
4. Repository context.

If the required ticket information cannot be determined, ask the user.

Do not invent ticket numbers.

### Description

The description should:

* Be concise.
* Describe the actual change.
* Use lowercase.
* Use underscores instead of spaces.
* Avoid unnecessary words.
* Avoid punctuation other than underscores.

For example:

```text
PRJ-1234/PRJ-567-fix-null-device-state
```

is preferred over:

```text
PRJ-1234/PRJ-567-fix-the-problem-with-the-device-state-being-null
```

---

## Existing Branch Handling

Before creating a new branch, check whether the appropriate branch already exists:

```bash
git branch -a
```

If the user is already on a correctly named feature branch, continue using it unless they explicitly request a new branch.

If the corresponding branch exists remotely, do not create a duplicate branch. Determine whether the existing branch is the user's work and continue from it.

Do not overwrite another person's branch without explicit instruction.

---

## Change Review

Before committing, inspect the complete change:

```bash
git status
git diff
git diff --cached
```

Also inspect relevant files when necessary to understand the change.

Do not blindly commit unrelated modifications.

If the working tree contains unrelated changes, preserve them and commit only the changes belonging to the requested work.

Before committing, verify:

* The intended files are changed.
* No secrets, credentials, tokens, generated private files, or unrelated changes are included.
* The diff matches the user's requested work.
* Tests/lint/type checks appropriate to the repository are run when practical.

---

## GitLab Identity

All commits, pushes, and merge requests created by this skill must be
associated with the user's own GitLab identity.

The user's GitLab identity is required because the repository's CI pipeline
uses the author/creator identity to determine whether CI should trigger.

### Commits

Commits must be created under the user's configured Git identity.

Before committing, verify:

```
git config user.name
git config user.email
```

Do not change the user's Git identity unless explicitly instructed.

Do not use another developer's identity, a shared/service account identity,
or an automated/bot identity for commits.

### Pushes

Push the branch using the user's own GitLab credentials/account.

The push must not be performed using another user's GitLab account or a
shared/service account.

The normal push destination remains the user's fork:

```
git push -u origin <feature-branch>
```

### Merge Requests

Create the merge request using the user's own GitLab account.

The merge request must show the user as its author/creator.

Do not create the merge request using:

* Another developer's account.
* A shared GitLab account.
* A service account.
* A bot account.

This requirement applies even when the repository or GitLab CLI is already
authenticated as another account. If the current GitLab authentication
cannot create the MR under the user's identity, do not silently create it
under another identity.

### CI Requirement

Treat user identity as a functional requirement, not merely metadata.

The expected workflow is:

```
User's Git identity
      ↓
User's commit
      ↓
User's fork
      ↓
User's GitLab identity
      ↓
User-created merge request
      ↓
CI pipeline triggers
```

If the push succeeds but the resulting MR is not associated with the user's
GitLab identity, investigate the identity/authentication configuration before
considering the operation complete.

---

## Commit

Create a concise commit message describing the actual change.

Prefer conventional repository-specific commit conventions when the repository already uses them.

If the repository uses ticket-prefixed commits, follow that convention.

Do not create artificial or misleading commit messages.

Example:

```text
PRJ-567 Fix audio subscription cleanup
```

Before committing:

```bash
git status
git diff
```

Then commit only the intended changes.

---

## Push

Push the feature branch to the user's fork:

```bash
git push -u origin <feature-branch>
```

Do not push the feature branch to `upstream` by default.

If the user explicitly requests another remote, follow that instruction.

If pushing fails because the remote branch has diverged, do not force-push automatically.

Inspect the divergence and determine the safest next step.

Never use:

```bash
git push --force
```

unless the user explicitly authorizes rewriting the remote branch.

---

## Merge Request

After pushing, create a merge request from:

```text
<user-fork>:<feature-branch>
```

into:

```text
<canonical-repository>:<base-branch>
```

The merge request's target must be the base/release branch determined earlier.

Do not automatically target:

```text
main
master
develop
```

unless that is actually the appropriate base branch.

### Merge request title

Use a concise title describing the change.

Prefer including the ticket identifier when consistent with repository conventions.

Example:

```text
PRJ-567 Fix audio subscription cleanup
```

### Merge request description

Include:

* What changed.
* Why it changed.
* Relevant implementation details.
* Testing performed.
* Any known limitations or follow-up work.

Keep the description factual and concise.

Do not claim tests were run if they were not.

---

## GitLab / GitHub CLI

Determine which platform the repository uses from its remote URLs and repository configuration.

For GitLab, prefer the repository's configured GitLab CLI tooling when available.

For GitHub, prefer the repository's configured GitHub CLI tooling when available.

Before creating an MR/PR, verify the command will target:

* The user's fork as the source repository.
* The user's feature branch as the source branch.
* The canonical repository as the target repository.
* The detected release/base branch as the target branch.

If CLI tooling is unavailable, provide the appropriate web URL for creating the MR/PR rather than pretending it was created.

---

## Safety / Confirmation Rules

Do not ask for confirmation for ordinary operations that are explicitly requested by the user, such as:

* Creating the feature branch.
* Committing the requested changes.
* Pushing the feature branch to the user's fork.
* Creating the merge request.

Do ask for clarification when:

* The ticket number is missing or ambiguous.
* The correct base/release branch cannot be determined.
* Multiple plausible release branches exist.
* The destination fork/remote is ambiguous.
* The changes contain unrelated modifications that cannot safely be separated.
* Pushing would require rewriting remote history.
* The requested operation would modify or delete someone else's work.

Never:

* Push directly to a release branch by default.
* Force-push without explicit authorization.
* Invent ticket numbers.
* Invent a base branch.
* Claim an MR/PR was created when it was not.
* Include unrelated working-tree changes merely because they are present.

---

## Expected Workflow

For a normal request, follow this sequence:

```text
Inspect repository
    ↓
Identify fork and upstream remotes
    ↓
Determine ticket identifiers
    ↓
Determine release/base branch
    ↓
Fetch latest base branch
    ↓
Create/switch to feature branch
    ↓
Implement or review requested changes
    ↓
Run appropriate validation
    ↓
Review git diff
    ↓
Commit
    ↓
Push feature branch to fork
    ↓
Create MR/PR targeting detected base branch
    ↓
Add web URl of the MR/PR to JIRA ticket if it is not already present
    ↓
Report result
```

At the end, report:

```text
Base:     <base branch>
Branch:   <feature branch>
Remote:   <fork remote>
Commit:   <commit hash/message>
MR/PR:    <URL>
```

If any step was not completed, state exactly which step failed and why.
