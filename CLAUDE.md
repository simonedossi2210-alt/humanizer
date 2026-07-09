# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **Claude Code / OpenCode skill** ("humanizer") implemented entirely as Markdown. The runtime artifact is `Skills/SKILL.md`: the agent reads its YAML frontmatter (name, version, description, allowed tools) followed by the editor prompt body, which turns the agent into a writing editor that removes signs of AI-generated text. There is no build step, no source code, and no tests to run — editing this repo means editing prose, not code.

## Repository layout

All tracked files live under `Skills/`:

- `Skills/SKILL.md` — the skill itself, and the **source of truth**. YAML frontmatter (`name`, `version`, `description`, `license`, `compatibility`, `allowed-tools`) followed by the canonical, numbered pattern list (33 patterns across Content, Language/Grammar, Style, Communication, and Filler/Hedging categories) with before/after examples, detection guidance (false positives to avoid, signs of genuine human writing to preserve), and the "Process and Output" workflow contract.
- `Skills/README.md` — human-facing docs: installation for Claude Code and OpenCode, usage examples (including voice calibration), a summary table mirroring the patterns in `SKILL.md`, and a version history.
- `Skills/AGENTS.md` — near-duplicate of this file's maintenance contract, targeted at non-Claude coding agents (Codex, Warp, etc.) that read `AGENTS.md` instead of `CLAUDE.md`.
- `Skills/CLAUDE.md` — a nested copy of this repo's Claude guidance, scoped to the `Skills/` directory itself (relevant if a session's working directory is `Skills/` rather than the repo root).
- `Skills/LICENSE` — MIT.

There is no root-level source beyond this file; everything a Claude Code or OpenCode installation actually loads (`SKILL.md`) lives one directory down, in `Skills/`.

## The maintenance contract

`Skills/SKILL.md` and `Skills/README.md` must stay in sync. When changing behavior or content:

- **Patterns:** the skill currently defines 33 numbered patterns. If you add, remove, or renumber any, update in the same change: the README pattern table, its "N Patterns Detected" heading, and every cross-reference. Keep numbering stable unless deliberately renumbering.
- **Version:** `SKILL.md` frontmatter has a `version:` field; `README.md` has a "Version History" section. Bump both together, in the same change.
- **Non-obvious fixes:** if you change the prompt to handle a tricky failure mode (a repeated mis-edit, an unexpected tone shift), add a short note to the README version history explaining what was fixed and why.
- **Three guidance files:** `AGENTS.md` and the nested `Skills/CLAUDE.md` mirror this file's contract for other tools/working directories. If you update this maintenance contract, update those two as well so all three stay consistent.

## Editing SKILL.md

- Preserve valid YAML frontmatter (formatting and indentation matter).
- The prompt below the frontmatter is the product, not code — edit it like a careful instruction document. Each numbered pattern follows the same shape: words/phrases to watch, the problem, a Before example, an After example.
- The "Process and Output" section at the end of `SKILL.md` defines the actual workflow the skill executes (draft rewrite → "what makes this obviously AI generated?" self-audit → final rewrite → deliverable). Don't change this contract casually — README usage instructions assume it.
- The "Detection Guidance" section (false positives, signs of human writing) exists to keep the skill from over-editing legitimate prose. Treat it with the same care as the pattern list — it's part of the contract, not an aside.
