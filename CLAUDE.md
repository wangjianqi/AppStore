# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**AppStore-Guide** is a documentation-only tutorial repository (no buildable code). It contains 10 sections (142 chapters + 12 appendices) guiding developers from zero to publishing an iOS app on the App Store, with an emphasis on AI-driven development workflows.

The content is written in Chinese and targets mainland China developers specifically (ICP filing, App Store review nuances, etc.).

## Repository Structure

- `01-环境与工具准备/` through `10-多平台开发/` — Main tutorial chapters (Markdown files)
- `附录/` — Appendices (FAQ, cheat sheets, checklists)
- `ios-claude-skills/` — 20 reusable Claude Code SKILL.md files for iOS development
- `.github/` — Issue/PR templates

There is no build system, test suite, or runnable code. All content is Markdown.

## Working with This Repo

### Content editing

- Files use Markdown with Chinese filenames in `标题.md` format (e.g., `AI编程工具全景.md`). Chapter order is defined by README.md table.
- Each chapter should open with a goal statement and close with a summary + navigation links
- Code examples go in fenced code blocks with language tags
- Use tables for comparisons, and emoji markers (💡提示, ⚠️警告) for callouts

### Commit conventions

Follow Conventional Commits:
- `feat:` new content
- `fix:` corrections
- `docs:` documentation changes
- `style:` formatting
- `refactor:` restructuring

Branch prefixes: `fix/`, `feature/`, `improve/`, `translate/`

### ios-claude-skills directory

Contains 20 Claude Code skill files organized by domain. Each `SKILL.md` has `name` and `description` frontmatter for trigger matching. These are designed to be copied into a project's `.claude/skills/` directory. The skills assume a UIKit + SnapKit + MVVM + SPM + StoreKit 2 stack (not SwiftUI).

## Key Technical Context

- Swift 5.9+, SwiftUI 5.0+, Xcode 15+
- AI tools covered: Claude Code, GitHub Copilot, Cursor, Trae, OpenAI Codex
- Core methodology: Spec-driven development + MCP protocol integration
