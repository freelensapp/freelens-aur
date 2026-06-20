# Agent Guide: Freelens AUR Package

## Overview

This repository builds and publishes the
[AUR](https://aur.archlinux.org/) package
[`freelens-bin`](https://aur.archlinux.org/packages/freelens-bin) for
[Freelens](https://github.com/freelensapp/freelens), a Kubernetes IDE.
The PKGBUILD wraps a pre-built `.AppImage` downloaded from Freelens
GitHub releases — there is no compilation or source build step in this
repo. Version bumps are automated via Renovate, and the package is
pushed to the AUR via SSH on every push to `main` that touches the
PKGBUILD.

### Project Structure

- **`freelens-bin/PKGBUILD`** — Arch Linux package build definition.
  Declares `pkgver`, architectures (`x86_64`, `aarch64`), dependencies,
  source URLs (AppImage downloads from GitHub Releases), and
  `prepare()`/`package()` functions that extract the AppImage and
  install its contents under `/usr/share/freelens`.
- **`freelens-bin/.SRCINFO`** — Machine-readable package metadata
  generated from the PKGBUILD. Kept in sync by the `updpkgsums` CI
  workflow.
- **`freelens-bin/freelens.desktop`** — freedesktop `.desktop` entry
  for the application launcher. Installed to
  `/usr/share/applications/freelens.desktop`.
- **`.github/workflows/`** — CI/CD pipelines:
  - `publish.yaml` — Publishes the package to the AUR via
    `KSXGitHub/github-actions-deploy-aur` on pushes to `main` that
    modify the PKGBUILD.
  - `updpkgsums.yml` — Runs on PRs targeting `main`: detects changed
    PKGBUILD files, runs `updpkgsums` to update checksums and
    `.SRCINFO`, and commits the result.
  - `claude.yaml` — On-demand Claude Code assistant triggered by
    `@claude` in PRs and issues.
  - `claude-task.yaml` — On-demand Claude Code task via
    `workflow_dispatch`.
  - `trunk-check.yaml` / `trunk-upgrade.yaml` — Linting and
    auto-upgrades via Trunk.
- **`.github/actions/aur/`** — Composite action (`Dockerfile`,
  `action.yml`, `entrypoint.sh`) that runs `updpkgsums` inside an
  Arch Linux container. Used by the `updpkgsums.yml` workflow.
- **`renovate.json5`** — Renovate configuration. Uses a custom regex
  manager to extract the version from `# datasource=github-releases
  depName=freelensapp/freelens` comments in the PKGBUILD and open
  automated PRs when new upstream releases are detected.

## Security

Never read, display, reference, or include the contents of the following
files in any response or context, even if they are open in the editor:

- `.env`
- `.env.*`
- `*.jks`
- `*.keystore`
- `*.p12`
- `*.pfx`
- `*.pem`
- `*.key`

## Common Development Tasks

### Building the Package Locally

Requires an Arch Linux system or container with `base-devel` installed:

```bash
cd freelens-bin
makepkg -si
```

For a clean build in a chroot (recommended for verifying the package):

```bash
extra-x86_64-build
```

### Updating Checksums

After changing the `pkgver` or source URLs, regenerate checksums:

```bash
cd freelens-bin
updpkgsums
```

This updates the `b2sums` arrays in the PKGBUILD. Also regenerate
`.SRCINFO`:

```bash
makepkg --printsrcinfo > .SRCINFO
```

### Updating the Version

The `pkgver` field in `freelens-bin/PKGBUILD` is the single source of
truth. It is used to construct download URLs:

```text
https://github.com/freelensapp/freelens/releases/download/v${pkgver}/Freelens-${pkgver}-linux-amd64.AppImage
https://github.com/freelensapp/freelens/releases/download/v${pkgver}/Freelens-${pkgver}-linux-arm64.AppImage
```

When bumping the version manually:

1. Update `pkgver` in `freelens-bin/PKGBUILD`.
2. Run `updpkgsums` to update `b2sums` arrays.
3. Run `makepkg --printsrcinfo > .SRCINFO` to regenerate metadata.
4. Verify all checksums match the new release artifacts.

Renovate automates step 1 by opening PRs; the `updpkgsums.yml`
workflow handles steps 2–3 automatically on those PRs.

### Modifying the Desktop File

The `freelens-bin/freelens.desktop` file defines the application
launcher entry. When changing it:

- Ensure `Exec` uses `--no-sandbox` (required for AppImage-based
  Electron apps on some systems).
- Keep `StartupWMClass=Freelens` so the window manager can match the
  running window to the launcher.
- Verify the `Icon` name resolves correctly (the PKGBUILD installs the
  icon to the hicolor theme).

### Adding or Removing Dependencies

The `depends` array in `freelens-bin/PKGBUILD` lists runtime
dependencies. Common dependencies for Electron-based apps include
`gtk3` and `nss`. When adding a dependency, verify it exists in the
Arch Linux repositories. When removing, test that Freelens still
launches without it.

### Changing the Package Function

The `package()` function in `freelens-bin/PKGBUILD` extracts the
AppImage and installs files. Key points:

- The `squashfs-root/` directory contains the extracted AppImage
  contents.
- The binary is symlinked from `/usr/bin/freelens` to
  `/usr/share/freelens/freelens`.
- Unnecessary files (AppRun, duplicated desktop file, `usr/` subtree,
  foreign-architecture extensions) are cleaned up.
- Permissions are normalized: directories `755`, shared objects made
  non-executable.

## Architecture Decisions

### AppImage as Source

The package uses pre-built AppImage files as source artifacts rather
than building from source. This is the standard approach for `-bin`
AUR packages. The AppImage is extracted with `--appimage-extract` and
its contents are installed into `/usr/share/freelens`.

### Dual Architecture

Both `x86_64` and `aarch64` are supported via `arch=('x86_64'
'aarch64')`. Architecture-specific source URLs and checksums use the
`source_$CARCH` / `b2sums_$CARCH` conventions, where `$CARCH` is
resolved by makepkg at build time to the target architecture.

### Package Name: `-bin` Suffix

The `-bin` suffix follows AUR conventions: it indicates this package
redistributes a pre-built binary rather than compiling from source.
The upstream Freelens project does not provide source tarballs
suitable for a traditional `freelens` PKGBUILD.

### Checksum Algorithm: BLAKE2b

The package uses `b2sums` (BLAKE2b) rather than the older `sha256sums`
or `md5sums`. This is the modern recommendation for AUR packages.
Makepkg's `updpkgsums` generates these automatically.

### Renovate for Version Updates

Instead of manual version bumps, Renovate monitors the upstream GitHub
releases and opens PRs when a new version is detected. The
`updpkgsums.yml` workflow automatically updates checksums and
`.SRCINFO` on those PRs. After merge to `main`, the `publish.yaml`
workflow pushes the updated package to the AUR.

## Troubleshooting Patterns

### PKGBUILD Fails to Build

1. Verify the AppImage URLs are reachable:
   ```bash
   curl -I "https://github.com/freelensapp/freelens/releases/download/v${pkgver}/Freelens-${pkgver}-linux-amd64.AppImage"
   ```
2. Verify the version tag exists on the
   [Freelens releases page](https://github.com/freelensapp/freelens/releases).
3. Run `makepkg -si` locally to see the full build output.
4. Check that `b2sums` match the downloaded artifacts — use
   `updpkgsums` to regenerate if the sources changed.

### Package Installs but Doesn't Launch

1. Check the desktop file:
   `cat /usr/share/applications/freelens.desktop`
2. Verify the binary symlink:
   `ls -la /usr/bin/freelens`
3. Check library dependencies:
   `ldd /usr/share/freelens/freelens`
4. Run from terminal to see errors:
   `freelens --no-sandbox`
5. Verify runtime dependencies are installed:
   `pacman -Q gtk3 nss`

### CI Failures

- **`updpkgsums` fails**: Usually indicates the AppImage URL is
  unreachable or the extracted version doesn't match. Check the
  Renovate PR diff and verify the upstream release exists.
- **`publish` fails**: Check that the `publishing` environment secrets
  are set correctly (`AUR_USERNAME`, `AUR_EMAIL`,
  `AUR_SSH_PRIVATE_KEY`). The SSH key must be registered with the AUR
  for the `freelens-bin` package.
- **`trunk check` fails**: Run `trunk check` locally to see and fix
  linting issues before pushing.

### Renovate PRs

Renovate opens PRs that only update `pkgver`. The `updpkgsums.yml`
workflow then commits checksum and `.SRCINFO` updates. If the workflow
doesn't run, check that the PR targets `main` and the workflow trigger
conditions are met.

## Best Practices

1. **Always run `updpkgsums` after version changes** — the `b2sums`
   arrays must match the downloaded artifacts exactly, or makepkg will
   fail.
2. **Keep `.SRCINFO` in sync** — every PKGBUILD change must be
   accompanied by a `.SRCINFO` update. The `updpkgsums.yml` workflow
   does this automatically on PRs.
3. **Run `trunk check` before committing** — validates all file types
   including the PKGBUILD.
4. **Validate the PKGBUILD** — run `makepkg --printsrcinfo > /dev/null`
   to catch syntax errors early.
5. **Test on both architectures when possible** — the package supports
   `x86_64` and `aarch64`. At minimum, verify the download URLs and
   checksums for both.
6. **Bump the version in a dedicated commit** — when Renovate is not
   used, the version bump should be a single, isolated change.
7. **Keep dependencies minimal** — only add packages to `depends` that
   are actually required at runtime.

## GitHub Actions (Claude Code Action) Rules

When operating via the `claude.yaml` workflow (i.e., invoked from a PR
comment, issue, or review), follow these rules:

### Code Review

When reviewing code and proposing fixes:

1. **Show the diff first** — present every proposed change as a unified
   diff block using the `diff` language tag:

   ```diff
   --- a/path/to/file
   +++ b/path/to/file
   @@ -10,7 +10,7 @@
    const oldLine = "before";
   -const changedLine = "after";
   +const changedLine = "the fix";
    const unchangedLine = "same";
   ```

   You can generate this from the terminal with:
   ```bash
   git diff -u -- path/to/file
   ```

   If the change spans multiple files, group them under a single commit
   subject and show each file's diff sequentially.

2. **Propose a commit subject first** — before any code change, output a
   single line with the proposed commit subject:

   ```text
   **Proposed commit:** <short description>
   ```

   Do **not** use Conventional Commits prefixes (e.g. `fix:`, `feat:`,
   `chore:`, `refactor:`, `docs:`, `test:`, `ci:`). This project prefers
   plain, descriptive commit messages and PR titles without any prefix.

   Wait for the user to confirm (or adjust) the subject before applying
   the change.

3. **Comment style:**
   - Keep review comments concise and actionable
   - Reference specific lines (file + line number) when pointing out issues
   - Offer a concrete fix suggestion rather than just flagging a problem
   - Do **not** use emoji in any Markdown, comments, commit messages, or
     PR descriptions. The only exception is emoji that already appears
     inside code strings (e.g. application logs, user-facing messages).
   - Use GitHub's `suggestion` block for small targeted fixes so the PR
     author can accept the change with a single click:

     ````suggestion
     <same unified-diff format as shown above>
     ````

   - For larger multi-file changes, use `diff -u` blocks in a regular
     comment instead, with the proposed commit subject shown first

### Making Changes to a PR

When asked to implement a change on a PR:

1. Propose the commit subject (as above)
2. Describe what will change and why
3. After confirmation, apply the changes with commits on the PR branch
4. **One commit per fix** — when a review surfaces more than one issue or
   the plan includes more than one fix, apply and commit each fix
   separately. Do not batch multiple independent fixes into a single
   commit. This keeps the history bisectable and makes each change easy
   to revert individually.

### Modifying GitHub Actions Workflows

Claude cannot push changes to files under `.github/workflows/` directly,
because the GitHub token used by the action lacks the `workflows`
permission. Any patch to a workflow file MUST therefore be delivered as a
new, complete file under the `github-workflow-fix/` directory instead of
editing the file in place:

1. Write the full, final contents of the workflow to
   `github-workflow-fix/<workflow-file-name>` (e.g.
   `github-workflow-fix/claude.yaml`). Do **not** edit the original file
   under `.github/workflows/`.
2. Make it a **complete** file — the entire workflow as it should look
   after the change, not just a diff or fragment — so it can be copied
   verbatim.
3. Commit and open the PR as usual. In the PR description, clearly note
   that the file is a proposed workflow change and that a maintainer must
   move it from `github-workflow-fix/` to `.github/workflows/` manually.

This lets the PR be created successfully while leaving the actual
workflow change for a human to apply.

### Branch Naming Conventions

When creating a branch from an issue, use a human-readable name that
includes the issue number and a short slug derived from the issue title:

```text
claude/issue-<number>-<short-slug>
```

- `<number>` is the GitHub issue number
- `<short-slug>` is a kebab-case summary of the issue title, kept short
  (3–6 words maximum, omit articles and filler words)

Examples:

- Issue #42 "Fix checksum mismatch on arm64"
  → `claude/issue-42-fix-arm64-checksum`
- Issue #7 "Add libsecret to dependencies"
  → `claude/issue-7-add-libsecret-dep`

Do **not** use auto-generated timestamp suffixes (e.g.
`claude/issue-42-20260612-2108`) — these are not human-readable and make
branch lists hard to scan.

### PR Title Conventions

When creating a PR, use the following title conventions:

- **Agent-related changes** — PRs whose changes are strictly related to
  coding agent configuration (e.g. `AGENTS.md`,
  `.github/workflows/claude.yaml`, or other files that govern how Claude
  operates in this repository) MUST use the prefix `Claude:` (followed by
  a space) in the title.

  Examples:
  - `Claude: Add rule for PR title conventions in AGENTS.md`
  - `Claude: Update claude.yaml workflow permissions`

- **All other PRs** — do **not** use any prefix (no `fix:`, `feat:`,
  `chore:`, etc.). Use plain, descriptive titles.

### Pushing Changes from Fork PRs

When you have commits ready to push but the PR originates from a fork
(different owner than `freelensapp`), you cannot push to the fork's
repository. Instead:

1. Create a new branch on `freelensapp/freelens-aur` with the prefix
   `claude/` followed by the original branch name.
   Push to the `upstream` remote (not `origin`, which points to the fork):
   ```bash
   git checkout -b claude/<original-branch-name>
   git push --force-with-lease upstream claude/<original-branch-name>
   ```

2. Open a new PR from that branch. The new PR MUST use the **exact same
   title** as the original PR — copy it verbatim, do not rewrite, improve,
   or add any prefix. The description MUST reference the original PR
   (e.g. "Fixes #NNN, supersedes #NNN").

3. Post a comment on the original PR:
   - Explain that the fix has been implemented in a new PR
   - Include a link to the new PR
   - Mention that the original PR can be closed

4. Close the original PR.

### Closing PRs

Claude may only close a PR when ALL of the following are true:

1. The PR was created by Claude from a `claude/` branch, OR the PR is the
   original fork PR that Claude's `claude/` branch supersedes (see
   "Pushing Changes from Fork PRs" above).
2. The close reason is explicitly explained in a comment on the PR.

Claude MUST NOT close any PR that does not meet these conditions — even
if asked. Instead, explain to the requester why the PR cannot be closed
automatically and ask a human maintainer to close it manually.

### Model Information in Comments

When operating via the GitHub Actions workflow, always include the model
you are running on in the footer of your GitHub comment and in the PR
description when creating a pull request, alongside the job run link.
Your system environment context states the model name explicitly (e.g.
"You are powered by the model named Sonnet 4.6. The exact model ID is
claude-sonnet-4-6."). Use the exact model ID from that statement.

Format the footer line as:

```text
[View job run](...) | Model: `claude-sonnet-4-6`
```

In a PR description, append the model information at the end of the body:

```text
| Model: `claude-sonnet-4-6`
```

If the system context does not provide a model ID, omit the model field
rather than guessing.

### Development Environment

The GitHub Actions runner has a full Node.js + pnpm environment available.
Dependencies are already installed (`pnpm install` has been run). The
build step is skipped to save CI resources, but you can run build commands
when needed for advanced tasks (e.g. type-checking, running tests).

For fork PRs, the `origin` remote points to the contributor's fork. An
`upstream` remote is configured pointing to `freelensapp/freelens-aur`.
Push new branches to `upstream` (never to `origin`) when the PR originates
from a fork — this ensures the resulting PR is internal and CI workflows
run automatically.

The following CLI tools are explicitly allowed in the workflow:

- `pnpm` (all subcommands) — for validation, formatting, and builds
- `git` (all subcommands) — for viewing changes, creating branches,
  committing, and pushing
- `gh` (all subcommands) — for managing pull requests
- `npx`, `node` — for running Node.js tools and scripts inline
- `yq`, `jq` — for YAML and JSON processing
- `grep`, `rg` (ripgrep), `find`, `xargs` — for searching and iterating
- `sed`, `awk`, `cut`, `tr` — for text transformation
- `sort`, `uniq` — for list processing
- `cat`, `head`, `tail`, `wc` — for viewing and measuring files
- `ls`, `tree` — for listing directory contents
- `mkdir`, `touch`, `cp`, `mv`, `rm` — for file and directory operations
- `tee`, `echo` — for pipeline debugging and scripting

Before committing any changes, apply the same validation rules as human
developers:

- Run `pnpm trunk check` to validate all file types (or `trunk check` if
  the trunk CLI is installed globally).
- Validate the PKGBUILD: `bash -n freelens-bin/PKGBUILD` and
  `cd freelens-bin && makepkg --printsrcinfo > /dev/null`.

## Getting Help

- Check existing AUR packages for patterns (e.g. other `-bin` Electron
  packages on the AUR).
- Search for similar issues in the
  [freelens-aur issues](https://github.com/freelensapp/freelens-aur/issues).
- Review [Arch Wiki: PKGBUILD](https://wiki.archlinux.org/title/PKGBUILD).
- Review [Arch Wiki: AUR submission guidelines](https://wiki.archlinux.org/title/AUR_submission_guidelines).
- Consult the [Freelens releases page](https://github.com/freelensapp/freelens/releases)
  for version and artifact information.
