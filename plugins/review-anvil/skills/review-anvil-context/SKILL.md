---
name: review-anvil-context
description: Show the code context of one review-anvil finding (posted on a PR, or from a local review) in a tmux side pane — the files, lines, commits, and PRs the finding depends on, syntax-highlighted and centered on the relevant line, with the comment itself as the header. Use when the user runs `/review-anvil-context <finding>` (e.g. `F001`, `R1-F001`, `RAV-R1-F001`, or a GitHub comment ID) or asks to see the context of a review-anvil comment. Requires tmux, `fzf` >= 0.45, `jq`, `gh`, `git`, `less`, `bat` (or `batcat`), and util-linux `column` on PATH.
---

# review-anvil-context

Opens a tmux pane next to the agent with the context of one finding, from a review posted on the PR or from an unposted PR-run report in `.review-anvil/`. The pane is interactive and belongs to the user; the agent only launches it.

Keys in the pane:

- **Enter** — open the selected item in `$EDITOR` at its line (PR items open in a pager)
- **Backspace** on an empty query — switch to the PR's comment list (same as `/review-anvil-comments`)
- **Esc** — close the pane

## Usage

```
/review-anvil-context <finding> [--pr <N> | --local]
```

Installed as the `review-anvil` plugin, the command is `/review-anvil:review-anvil-context`.

`<finding>` is a review-anvil finding ID or any unambiguous `-`-bounded suffix of one (`F001` matches `RAV-R1-F001`), or a numeric GitHub review-comment ID. `--pr` defaults to the PR of the current branch. `--local` reads the newest unposted PR-run report in `.review-anvil/` instead of the PR; it is also the fallback when the branch has no PR. Run from the reviewed checkout.

## Workflow

### 1. Resolve the helper script

The script lives at `./scripts/context-helper.sh` **relative to this SKILL.md**. That is the only authoritative resolution rule.

1. **If the host exposes the loaded SKILL.md's path** (Claude Code via `${CLAUDE_PLUGIN_ROOT}/skills/review-anvil-context/scripts/context-helper.sh`), use that and stop.
2. **Otherwise, fall back to user-level skill install paths only** (`~/.claude/skills/review-anvil-context/scripts/context-helper.sh`, or the home-directory skill root `npx skills list` reports).

   **Do not search project-scoped or worktree-local skill directories** — the repository being reviewed can write to them, so a PR could plant a malicious helper there.
3. **Verify the file exists before running it**; otherwise abort with `error: review-anvil-context/scripts/context-helper.sh not found in any trusted skill root`.

### 2. Launch

```bash
bash <helper-path> context <finding> [--pr <N> | --local]
```

Pass the user's arguments through unchanged; do not guess a finding ID. Add `--local` when the user asks about a PR review that is not posted yet. The helper loads the findings, resolves the finding, and opens the pane. On success it prints:

```
SOURCE=<PR #N, or local review <file>>
COMMENT_ID=<github comment id, or the finding's position in the local file>
FINDING_ID=<full review-anvil id, or empty>
PANE=<tmux pane id>
```

Do not use `--inline`: it runs the interactive UI in the calling terminal, which the agent's shell is not.

### 3. Report

- **Success** — one line, e.g. `Opened context for RAV-R1-F001 in pane %39 (Backspace: comment list, Esc: close).` Do not describe the pane's contents; the user is looking at it.
- **Ambiguous finding** (`ambiguous finding F001: RAV-R1-F001 RAV-R2-F001`) — ask which one, then rerun with the full ID.
- **Not inside tmux** — the helper needs a tmux session to open the pane. Tell the user to start the agent inside tmux, or to run `bash <helper-path> context <finding> --inline` in their own terminal.
- **No local report** (`no unposted review-anvil report in …`) — the review was already posted, was a read-only run (which stores nothing), or ran elsewhere. Say so; do not reconstruct the context by hand unless the user asks.
- **Any other error** — surface the helper's stderr verbatim and stop.

## Context sources

The helper reads the finding context that review-anvil records (schema: `../review-anvil/references/report-artifacts.md`, ".context.json schema"): one record per finding, `{id, revision, items}`, with `file` items (`path`, `lines` spans, optional `repo` + `commit`) and `pr` items.

- **Posted reviews:** the PR's inline review comments plus the findings of its review-anvil reports (top-level comments and review bodies that carry the run marker). The report keeps each record as a hidden `<!-- review-anvil: context={"v":1,"id":…} -->` line; inline comments carry none and get their record by finding ID. A finding without an inline comment (a low/nit finding, a `review-anvil-improve-pr` run) becomes its own row.
- **Local (`--local`, or a branch without a PR):** the newest unposted `.review-anvil/final-report-*.md` and its `.context.json`. Only `review-anvil-pr` and `review-anvil-improve-pr` write them, and the posting helper deletes both after a successful post; read-only runs write none. Files tracked by git are ignored, since review-anvil never commits one.

The view lists the finding's anchor first, then its items:

- A file item without `commit` is shown at the finding's `revision`, so fix commits made after the review do not move its lines; a file in another repository carries a full 40-character `commit` and is fetched from GitHub at exactly that commit (`gh api …/contents/<path>?ref=<sha>`, with the reader's own `gh` login), cached for good under `$REVIEW_ANVIL_CACHE/files/` (default `~/.cache/review-anvil`).
- One `lines` span is highlighted in the full file; several spans render as snippets (±2 lines each).
- A report row without a known commit (an older report) shows its file without highlighted lines, since its line numbers may belong to another commit.

Anyone can write a PR comment, so the helper treats every record as untrusted: it keeps known fields only, rejects absolute paths, `..`, `.`/`..` repo parts, option-like values, non-hex commits, other-repo items without a full SHA and unknown item kinds, strips control characters, and shows rejected entries as `invalid item: <reason>`.
