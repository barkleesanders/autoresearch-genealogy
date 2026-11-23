# Setup prompt — hand this to your agent

Copy everything in the block below into a fresh session with your coding agent
(Claude Code, Codex, or any agent that can run shell commands and edit files).
It will install the whole stack: the genealogy skill, the decompile skill, and
the fcdp browser bridge, then create your private vault and verify each piece
with evidence — not just claim it works.

---

```text
Set up the autoresearch-genealogy research stack for me, from this repo:
https://github.com/barkleesanders/autoresearch-genealogy

Work through these steps in order. After each step, verify it with a real
command and show me the output. Do not tell me a step is done without evidence.

1. Clone the repo to a permanent location (not /tmp).

2. Install the two skills into your skills directory (e.g. ~/.claude/skills/):
   - SKILL.md at the repo root -> <skills>/autoresearch-genealogy/SKILL.md
   - skills/decompile/ (the whole folder) -> <skills>/decompile/
   Confirm both files exist and print their first lines.

3. Install the browser bridge from tools/fcdp/:
   - Copy tools/fcdp/ to a permanent location (e.g. ~/tools/fcdp/).
   - Start the bridge: node bridge.mjs (set it up as a background service
     if my system has one — launchd on macOS, systemd --user on Linux).
   - I will load the extension myself: tell me to open chrome://extensions,
     enable Developer mode, choose "Load unpacked", and select the
     extension/ folder inside the copied tools/fcdp/. Wait for me to
     confirm it's loaded before continuing.
   - Label the profile: run `fcdp profiles`, then `fcdp label <conn> "Profile 1"`
     (or the directory name of the profile I loaded it in).
   - Verify end to end: `fcdp tabs` lists my real tabs; `fcdp open
     https://example.com` opens a tab; `fcdp text` prints the page text.
     Show me all three outputs.

4. Create my private research vault OUTSIDE the repo, in a location you
   propose and I approve, with the layout the genealogy skill describes:
   a tree file, an open-questions file, a research log, an evidence
   directory, and a prompts/workflows area. Nothing in the vault is ever
   committed, published, or shared.

5. Run one harmless end-to-end check: using the bridge, open a public
   genealogy search page (e.g. FamilySearch search) and report what loaded.
   Do not log in to anything, do not search for a real person yet.

Standing rules for the whole setup and everything after it:
- Never ask me for passwords and never handle them. I sign in to sites in
  my own browser; the bridge replays that session.
- Living people are never named in anything that leaves the vault.
- Nothing is sent, ordered, posted, or filed without my explicit approval
  of the exact content.
- If a step fails, show me the actual error and what you tried — do not
  silently switch to a different method.

When everything is verified, give me a short report: what was installed,
where, and the verification output for each piece.
```

---

## Why a prompt?

The method in this repo is only half the system. The other half is an agent
that installs it carefully, verifies each piece against reality, and keeps
your private research private. This prompt encodes that discipline: every
step ends in evidence, and the privacy rules are stated before any research
begins.
