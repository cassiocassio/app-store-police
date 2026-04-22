# app-store-police

An adversarial pre-flight review agent for Mac App Store submissions, for use with [Claude Code](https://docs.claude.com/en/docs/claude-code).

Three personas run in parallel on every review:

- **BOT** — the static analyser. Mechanical checks: `codesign --verify --strict`, `spctl --assess`, `stapler validate`, entitlements sanity, hardened runtime, Info.plist keys.
- **REVIEWER** — the human App Store reviewer. Reads the Review Guidelines and asks "would I reject this?" with the rejection boilerplate and the resubmit reply.
- **INDIE** — the scarred Mac indie veteran. Pattern-matches against real rejections (Python sidecars, sandbox migration, security-scoped bookmarks, bundled binaries, StoreKit gotchas) and tells you which fight to pick.

It answers one question:

> **Will this submission be accepted, and if not, why will it be rejected?**

It is **not** a UX or HIG reviewer. For taste and native feel — whether the app *feels* like a Mac app — use the sibling agent: **[what-would-gruber-say](https://github.com/cassiocassio/what-would-gruber-say)**. This one is about survival.

The split is deliberate. UX problems that cause rejection (a hidden menu bar, a broken reserved shortcut, a non-compliant sandbox configuration) land on *both* agents' desks; taste problems that won't get you rejected (a slightly off-beat sidebar pattern, an idiosyncratic toolbar layout) are Gruber's lane only. Install both if you're shipping to the Mac App Store.

## When to use it

- Before a TestFlight or App Store submission
- When touching entitlements, signing, `Info.plist`, or sandbox code
- When you want a cold read on "will this ship?"
- After a rejection, to draft a resubmit reply or decide whether to appeal

## Installation

Copy the agent into your project:

```bash
mkdir -p .claude/agents
curl -o .claude/agents/app-store-police.md \
  https://raw.githubusercontent.com/cassiocassio/app-store-police/main/.claude/agents/app-store-police.md
```

Or drop it into `~/.claude/agents/` for every project on your machine.

Then invoke it from Claude Code via the Task tool with `subagent_type: "app-store-police"`, or Claude will pick it up automatically when the task description matches.

## What it needs

The agent uses `Read`, `Glob`, `Grep`, `Bash`, `WebFetch`, and `WebSearch`. It runs mechanical checks via shell — `codesign`, `spctl`, `stapler`, `otool`, `plutil`, etc. — so run it on macOS with Xcode command-line tools installed.

It will `WebFetch` the current App Store Review Guidelines rather than relying on a snapshot, so findings cite live guideline text.

## Status

Lightly battle-tested on one Python-sidecar Mac app in progress. Expect the INDIE persona's war stories to drift as Apple's rules change — treat references (TN3125, Background Assets Framework, SMAppService) as starting points, not gospel. Pull requests welcome, especially for new rejection patterns.

## Licence

MIT. See [LICENSE](LICENSE).

Copyright © 2026 Martin Storey.
