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

## Shipping a bundled Python interpreter? Read this first

If your Mac app wraps a **PyInstaller / python-build-standalone / BeeWare / embedded-`Python.framework` sidecar** driven from a Swift host, the agent opens with a dedicated **"Python-sidecar dance — survival checklist"**: the entitlements matrix (which binary gets `app-sandbox` vs `inherit` vs `cs.*` vs resource keys), the extension-less framework-binary signing trap, the JIT entitlements numba/llvmlite force under Hardened Runtime, the runtime sandbox wounds (bare-name `ffmpeg` shellouts, `mimetypes` `PermissionError`), and the verification commands.

The hardest lesson it encodes: **a clean local `codesign --verify` run does not mean App Store Connect will accept the upload.** Nested-executable sandbox coverage, the framework `--identifier`, and the app category are all server-side-only rejections. The appendix is a worked case study of a real sidecar app's first-upload rejections (missing category, nested binaries without `app-sandbox`, an unsigned `Python` framework binary) and how each was fixed — plus the `ITMS-90296` / `ITMS-90287` config gates and the Xcode build-pipeline traps that stop you ever producing a valid upload.

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

Battle-tested taking a sandboxed Python-sidecar Mac app (Swift host + PyInstaller sidecar + bundled ffmpeg) through App Store Connect to internal TestFlight. The survival checklist and the appendix case study are drawn from that app's *actual* first-upload rejections and the fixes that cleared them — not from theory. Everything App-Store-Connect-specific (the server-side rejections, the `ITMS-90296`/`90287` gates, the build-pipeline traps) is empirical.

The signing/sandbox/manifest/§2.5.2 rules were then cross-checked against Apple's official docs and Apple DTS forum answers (a multi-source, adversarially-verified pass). Highlights folded in: the `itms-services` static-scan trap that auto-rejects bundled CPython 3.12+ (fix: `--with-app-store-compliance`); the `get-task-allow` auto-injection that crashes Debug nested helpers; library-validation requiring you to re-sign vendored wheels with *your* Team ID; the two distinct JIT entitlements (`allow-jit` MAP_JIT vs `allow-unsigned-executable-memory`) and why numba/llvmlite may need the broader one; and the 12 Feb 2025 privacy-manifest enforcement regime. Where a popular claim contradicts Apple's published text (e.g. "`inherit` + any other entitlement aborts the child" — it doesn't; only `get-task-allow` does), the agent now says so.

Expect the INDIE persona's war stories to drift as Apple's rules change — treat references (technotes, forum thread numbers, the named-SDK list) as starting points, and re-fetch the live pages rather than trusting a snapshot. Pull requests welcome, especially for new rejection patterns and for non-PyInstaller sidecar toolchains (BeeWare/Briefcase, python-build-standalone, PythonKit).

## Licence

MIT. See [LICENSE](LICENSE).

Copyright © 2026 Martin Storey.

---

## Releasing changes to this agent

This repo is a **release** cut from a working copy that lives inside a private
project, where the agent gets used on real work. That's what keeps it good — and
it's also how project-specific detail ends up in the file.

`.githooks/pre-push` is the gate that stops such detail reaching a public push.
It is tracked, but git does not enable it automatically. **After cloning, run:**

```sh
git config core.hooksPath .githooks
```

The gate screens tracked files case-insensitively and, if you keep a private
pattern file, augments its list at runtime — that file is never committed, since
publishing a list of terms you consider sensitive is itself a disclosure. Point
it somewhere else with `AGENT_LEAK_PATTERNS=/path/to/file`.
