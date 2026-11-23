# autoresearch-genealogy

An agent skill for AI-assisted genealogy research: family trees, ancestor hunting, DNA-match triangulation, and vital-records work — run with the source rigor of a professional engagement.

It works with any agent that can read a `SKILL.md`-style instruction file (Claude Code and compatible harnesses). The skill is the method — and this repo ships the tooling the method assumes, including the browser bridge below.

## What's in this repo

- **`SKILL.md`** — the genealogy research skill (the method).
- **`SETUP-PROMPT.md`** — a copy-paste prompt that makes your agent install and verify the whole stack for you. Start here.
- **`tools/fcdp/`** — the browser bridge the research runs on: a small Chrome extension + local daemon + CLI that expose most of the Chrome DevTools Protocol on your *real, logged-in* Chrome profile, so your agent can drive record and DNA sites through your own session. Localhost only; it never sees your passwords.
- **`skills/decompile/`** — a companion reverse-engineering skill: how to take apart binaries, apps, browser extensions — and the private APIs of websites you're logged into (HAR session capture → typed client). This is how the session-replay clients the genealogy method uses get built.

## Why this exists

Genealogy is where AI assistants hallucinate most confidently: same-name people get fused, index OCR gets quoted as fact, and "no results" gets reported as "this person didn't exist." This skill was built over dozens of long research sessions — one campaign carried a family's lines back nine generations and resolved a contested-parentage case from DNA plus records in about 27 hours of wall clock, against a published professional pace of "at least 26 hours of research over 3 to 4 months" for the same class of problem. The speed did not come from searching more. It came from discipline:

- **Negatives need positive controls.** Every search has three outcomes — found / genuinely absent (proven with a control hit outside the searched set) / could not measure — and the third is reported honestly instead of being silently upgraded to the second.
- **Read the image, not the OCR.** Census sheets, funeral ledgers, newspaper pages, and town reports all produce confident, wrong OCR. The skill makes the page image the record.
- **Use your own logged-in sessions as instruments.** Record sites run scriptable internal search APIs behind their web UI. A small local client that replays *your own* browser session turns them into batch instruments — with paging ceilings, hint-vs-filter parameters, and other silent failure modes measured and written down.
- **Confidence tiers on every claim**, a research log that records retractions instead of deleting them, and a hard rule that living people are never named outside the private vault.

## How it works

A research session under this skill runs a fixed protocol:

1. **Decompose** the objective into a few deliverables, each with the evidence that would settle it.
2. **Check instruments before searching** — session liveness for authenticated sites, browser bridge up.
3. **Research in passes** (registers, deaths, newspapers, DNA, other-country records), each pass writing its own findings file under a shared rules file; the main session merges and cites.
4. **Archive everything** into one evidence directory per session, with the query that produced each item.
5. **Log the session** — numbered findings with source ids, negatives with their controls named, instrument notes, and what was deliberately not done.
6. **Order records** only through a draft-shown-first, human-approves gate, with fees and forms re-verified against the custodian's own page.

The skill file itself documents the measured behavior of the major instruments: FamilySearch's internal search service (paging ceiling, ranking-hint parameters, tree-merge traps, coverage-bounded collections), DNA match triangulation (relatives-in-common clustering, generation placement by shared-match ratios), newspaper archives, vital and civil indexes, graves and funeral records, and the records-ordering table for common custodians.

It also includes a validated method for **DNA-driven adoption and contested-parentage searches**: cluster the parent's side, find the couple the sub-clusters converge on, build down by record, and keep the identity-bridge claim separate from the parentage claim.

## Privacy model

This is a method, not a dataset. It contains no one's family data.

- All research lives in a **private vault** on your machine; the skill file carries method only.
- **Living people are never named** in anything the workflow publishes; a pre-publication sweep (with a positive control proving the sweep works) is mandatory.
- The agent **never handles your passwords**: you sign in to record and DNA sites in your own browser; the tooling replays that session's cookies from a permission-locked local cache, and you re-sign-in when it expires.
- Nothing is sent, ordered, or posted without your explicit approval of the exact text or form.

## Install

**Fastest path:** give your agent the prompt in [`SETUP-PROMPT.md`](SETUP-PROMPT.md) — it installs and verifies everything, including the browser bridge.

By hand: copy `SKILL.md` into your agent's skills directory, e.g.:

```bash
mkdir -p ~/.claude/skills/autoresearch-genealogy
cp SKILL.md ~/.claude/skills/autoresearch-genealogy/
```

Then invoke it by name (`/autoresearch-genealogy`) or let your agent trigger it on genealogy topics. Create the vault layout described in the skill (tree, open questions, research log, evidence directories, prompts, workflows) wherever you keep private research — outside any published repository.

## What's not included

- **No hosted services, no accounts.** Everything runs locally: the bridge daemon listens on localhost and a Unix socket, the extension talks only to it, and your research data never leaves your machine.
- **No data.** No trees, no names, no DNA files. Yours stay in your vault.
- **No guarantees about platform behavior.** Instrument behaviors were measured in real sessions and are dated as such in the skill; platforms change, and the skill's own rule is to re-measure before relying on any of them.

## Contributing

The most valuable contributions are *measured instrument behaviors*: a platform trap you verified with a positive control, a search pattern that produced a record, a custodian rule you confirmed on the custodian's own page. Method only — never case data, never names of living people.

## License

MIT — see `LICENSE`.
