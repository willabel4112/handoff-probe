---
name: handoff-probe
description: Interview a handoff document with a fresh, ignorant agent before signing off. Use right after writing a HANDOFF / continuation / safepoint document for a successor session, when the user asks to "probe", "stress-test" or "interview" a handoff, or before ending a session whose work someone else (or a fresh context) will continue.
---

# handoff-probe — the veil-of-ignorance interview

You just wrote a handoff document. You cannot see which of your assumptions are non-obvious —
that is the curse of knowledge. This skill runs a cold, text-only probe that reads ONLY the
goal and the document, plans its first action, and asks the questions that would block it.
You answer, fold the answers into the document, and run it again until a fresh reader is dry.

## When

- After writing a handoff for **work** a successor must continue → kind `work` (default).
- After writing a document a successor must **understand** (retrospective, design notes,
  a knowledge dump with no first-action) → kind `knowledge`.
- A pure checkpoint (facts only, nobody will act on it) needs **no** probe.

## How (loop until dry)

1. Run it bare — no `timeout`, no pipe, no `&&` (wrappers break command allowlists; the
   deadline is built in):

   ```
   handoff-probe <handoff-file> "<goal of the next session>" [work|knowledge]
   ```
   (`bin/` of an enabled plugin is on the Bash PATH; the absolute form is
   `"${CLAUDE_PLUGIN_ROOT}"/bin/handoff-probe`.)

2. Read the output: `PLAN:` (or `SUMMARY:` in knowledge mode) then `QUESTIONS:`.
3. For each question, **answer from the repo / git log / notes — not from your memory of the
   session**. If the question rests on a false premise, note "false premise" and treat it as
   dry; the probe is rewarded for finding questions and will manufacture blockers if you follow it.
4. **Write the answers into the document** (this is the point — the doc improves, not the chat).
   Also check the `PLAN:`/`SUMMARY:` lines: a wrong interpretation there is a document ambiguity
   even if the probe did not phrase it as a question. Fix those too.
5. Run again. Every round is a fresh probe with no memory of the last one.
6. Stop after **2 consecutive dry rounds** (`NO-BLOCKING-QUESTIONS`) or about 5 rounds total.
   What is left is residue that only surfaces when the successor actually starts working.

## Cost and knobs

- One round ≈ a single small `claude -p` call (typically well under a dollar; the script prints
  the cost). The probe is cold and small; your own answers ride your warm context.
- `PROBE_MODEL` to pick a cheaper model for the probe · `PROBE_TIMEOUT` seconds (default 900) ·
  `PROBE_DIR` where the throwaway probe sessions live (default `~/.handoff-probe`, so they never
  appear in your real projects' resume pickers).
