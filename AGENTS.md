# Strimzi Proposals

## Scope

- This repo is a Markdown proposal archive, not a codebase. Proposal files live at the root as `NNN-title.md`; proposal assets live under `images/`.
- There is no root build, test, lint, formatter, or package manifest.

## Proposal Workflow

- Start new proposals by copying the `000-template.md`, but the template explicitly allows adding or removing sections to fit the proposal.
- Use the next free three-digit sequence number for a new proposal and all related assets before merge, matching `.github/PULL_REQUEST_TEMPLATE.md`.
- When opening a PR with an AI tool, always use `.github/PULL_REQUEST_TEMPLATE.md`.
- Do not update `README.md` when opening a proposal PR; update the index only at the end before maintainers merge the PR.
- `README.md` is the proposal index and is ordered newest/highest number first.
- Name proposal-specific images/assets with the same numeric prefix as the proposal, following existing files such as `images/144-...svg`.

### Proposal Format

- Proposal files must be Markdown
* You MUST follow a strict one sentence per line format
- Do NOT number proposal sections (e.g., "1. Motivation", "2. Proposal"), use Markdown headers only (`##`, `###`)
- DO use numbered lists within sections when showing sequential steps
- DO use bullet points for non-sequential items
- DO NOT use emojis as bullet points

## Verification

- With no configured checks, verify Markdown edits by reviewing the rendered structure, relative links, and image paths.
- Run `git diff --check` before handing off if whitespace-sensitive Markdown changes were made.

## Local Files

- `.gitignore` excludes assistant-local config directories/files (`.opencode/`, `.claude/`, `.mcp.json`, `.bob/`); do not edit or commit them for proposal work unless explicitly asked.
