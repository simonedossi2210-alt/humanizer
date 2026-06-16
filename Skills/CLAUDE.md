# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **Claude Code / OpenCode skill** ("humanizer") implemented entirely as Markdown. The runtime artifact is `SKILL.md`: the agent reads its YAML frontmatter (name, version, description, allowed tools) followed by the editor prompt body. There is no build step, no source code, and no tests to run.

## Key files

- `SKILL.md` — the skill itself, and the **source of truth**. YAML frontmatter (`name`, `version`, `description`, `license`, `compatibility`, `allowed-tools`) followed by the canonical, numbered pattern list (33 patterns) with before/after examples, detection guidance, and the process/output contract.
- `README.md` — human-facing docs: installation (Claude Code / OpenCode), usage, a summary table mirroring the patterns in `SKILL.md`, and a version history.
- `AGENTS.md` — guidance for AI coding agents editing this repo (this file folds its contents in).

## The maintenance contract

`SKILL.md` and `README.md` must stay in sync. When changing behavior or content:

- **Patterns:** the skill currently defines 33 numbered patterns. If you add, remove, or renumber any, update in the same change: the README pattern table, its "N Patterns Detected" heading, and every cross-reference. Keep numbering stable unless deliberately renumbering.
- **Version:** `SKILL.md` frontmatter has a `version:` field; `README.md` has a "Version History" section. Bump both together, in the same change.
- **Non-obvious fixes:** if you change the prompt to handle a tricky failure mode (a repeated mis-edit, an unexpected tone shift), add a short note to the README version history explaining what was fixed and why.

## Editing SKILL.md

- Preserve valid YAML frontmatter (formatting and indentation matter).
- The prompt below the frontmatter is the product, not code — edit it like a careful instruction document. Each numbered pattern follows the same shape: words/phrases to watch, the problem, a Before example, an After example.
- The "Process and Output" section at the end of `SKILL.md` defines the actual workflow the skill executes (draft rewrite → "what makes this obviously AI generated?" self-audit → final rewrite → deliverable). Don't change this contract casually — README usage instructions assume it.
