# handoff-probe

**Interview your handoff document with a fresh, ignorant agent before you leave.**

You wrote a handoff so the next session (a fresh context window, a colleague, a future you) can
continue the work. The problem: you cannot see which of your assumptions are obvious only to you.
That is the *curse of knowledge*, and it is why handoffs fail on the first action — "which host?",
"which of the two configs?", "the usual key — which one?".

`handoff-probe` spawns a small, cold agent that sees **only** the goal and the document text — no
tools, no repo, no memory of your session — and makes it do two things:

1. write the first-action plan it would execute after taking over, and
2. ask **only** the questions that would block that plan.

You answer (from the repo, not from memory), fold the answers into the document, and run it again.
Every round is a fresh probe with no memory of the previous one. You are done when a fresh reader is
dry: the probe prints `NO-BLOCKING-QUESTIONS` twice in a row.

```
$ handoff-probe HANDOFF.md "Make the nightly photo backup reliable again"
PLAN:
1. On the session host, locate the job and its log (crontab -l, systemctl list-timers), read the
   script to get the rsync target — LAN or offsite decides whether wake-on-LAN is even possible …
QUESTIONS:
NO-BLOCKING-QUESTIONS
--- handoff-probe: cost=$0.89 | fresh probe per round; a dry round prints NO-BLOCKING-QUESTIONS
```

Two full rounds against the toy document in [`examples/`](examples/) are checked in verbatim
([work mode](examples/probe-round-1.txt), [knowledge mode](examples/probe-knowledge-round-1.txt)).

## Install

As a Claude Code plugin (ships the script plus a skill that tells the agent *when* to run it):

```
/plugin marketplace add willabel4112/handoff-probe
/plugin install handoff-probe@willabel4112
```

Or just take the script — it is one file with no dependencies beyond `bash`, `python3` and an
agent CLI:

```
curl -fsSL https://raw.githubusercontent.com/willabel4112/handoff-probe/main/bin/handoff-probe -o ~/.local/bin/handoff-probe && chmod +x ~/.local/bin/handoff-probe
```

## Usage

```
handoff-probe <handoff-file> "<goal of the next session>" [work|knowledge]
```

| kind | the successor must… | the probe asks about… | stop when |
|---|---|---|---|
| `work` (default) | **act** | what blocks its first action | 2 consecutive dry rounds, or ~5 rounds |
| `knowledge` | **understand** | what it cannot reconstruct from the text | same |

Pick the kind explicitly. A work-shaped probe on a knowledge document (a retrospective, design
notes) has no stop criterion and rabbit-holes. A pure checkpoint — facts nobody will act on — needs
no probe at all.

**The rules on your side matter more than the script:**

- Answer from the repo, git log and notes — **never from your memory of the session**. The whole
  point is that the document, not your head, is what survives.
- A question built on a false premise gets the answer "false premise" and **counts as a dry round**.
  The probe is rewarded for finding questions; if you follow it, it will manufacture blockers.
- Read the `PLAN:` / `SUMMARY:` lines, not just the questions. A wrong interpretation there is a
  document ambiguity even when the probe did not phrase it as a question — fix it in the document.
- Call it bare. If your agent runs under a command allowlist, wrapping the call in `timeout …`, a
  pipe or `&&` turns a pre-approved command back into a permission prompt; the deadline is built in
  (`PROBE_TIMEOUT`, default 900 s) for exactly that reason.

Knobs: `PROBE_MODEL` (cheaper model for the probe) · `PROBE_TIMEOUT` · `PROBE_DIR` (where throwaway
probe sessions live, default `~/.handoff-probe`, so they never appear in your real projects' resume
pickers) · `PROBE_PMODE` (default `plan`, read-only).

## Backends

The default backend is Claude Code (`claude -p --output-format json`, which also reports the cost).
The protocol is text-in / text-out and nothing else is Claude-specific, so any agent CLI works:

```
PROBE_CMD="codex exec" handoff-probe HANDOFF.md "…"        # OpenAI Codex CLI
PROBE_CMD="my-harness ask" handoff-probe HANDOFF.md "…"    # anything that takes the prompt as one argument and prints text
```

`PROBE_BARE=1` adds `--bare` to the Claude call (no CLAUDE.md / hooks / skills / plugins / MCP
discovery — the strictest veil). It needs an API key; with a subscription login `--bare` reports
"Not logged in", so it is opt-in. Without it the probe still sees your user-level `CLAUDE.md`,
which for us has been acceptable: what must stay behind the veil is the *session*, not the house
rules.

We intend later releases to work across harnesses (Codex, OpenClaw, Hermes, …) as a matter of
course; this one is packaged as a Claude Code plugin only because that is where we run it daily.

## How we actually use it

Numbers from one operator's fleet, roughly two months, several machines:

- **139 probe rounds, $104 total, median $0.73 per round.** A round is one small cold call —
  typically ~2 minutes, zero tool calls, ~15 K output tokens. Your own answers ride your warm context.
- Typical convergence: a well-written handoff is dry in 2 rounds; a rushed one takes 4 and yields
  2–3 real holes. The first real run (a 600 K-token session's handoff) surfaced "which machine, which
  path" gaps we had not noticed and one genuine ambiguity the writer did not know was there.
- **Dry ≠ no yield.** In a knowledge-mode run the probe declared two rounds dry, but its own
  `SUMMARY:` misread a measured comparison value as a pass threshold and inferred the opposite
  conclusion. The misreading was a real document bug. Read the summary lines.
- Usage self-corrected: in month one we ran a probe for about every second human session; after
  we wrote down *when* a handoff is worth it at all (one unfinished thing → just resume; only probe
  when a fresh brain takes over), probe volume dropped by ~60%. The tool is cheap; the discipline
  around it is what saves money.

There is **no A/B test and no benchmark** behind any of this. It is one practice, kept because
handoffs stopped failing on the first action. If you try it — especially if it does *not* work for
your kind of handoff — please say so in [Discussions](../../discussions) or an issue. Comparisons,
counter-examples and better stop criteria are exactly what we want.

## Prior art and what is different

- **Handoff skills** that *write* the document
  ([portable-agent-kit](https://lobehub.com/skills/cdrguru-portable-agent-kit-claude-code-handoff),
  [claude-skills/handoff](https://alirezarezvani.github.io/claude-skills/skills/productivity/handoff/),
  [agentcookbooks/handoff](https://agentcookbooks.com/skills/handoff/)). This tool assumes you already
  have one and tests it.
- **Interview skills** that interview the *user* before coding
  ([neonwatty](https://neonwatty.com/posts/interview-skills-claude-code/)). This one interviews the
  *document*, with a reader who is deliberately kept ignorant.
- **Session forking** (`/fork`, `claude --resume <id> --fork-session`) answers questions by
  reviving the old context. That is the layer *after* this one: predictable gaps die here, cheaply,
  while the writer is still around; only unpredictable residue should need a fork.
- The ideas are old: Rawls' veil of ignorance (a reader who does not know what the writer knows is
  the only one who can judge the document fairly), the curse of knowledge (Camerer, Loewenstein &
  Weber 1989), and hallway usability testing. The contribution here is only the packaging: an
  isolated, stateless, read-only reader; a stop criterion; and the two rules that keep it honest
  (answer from the repo; false premise counts as dry).

## About this account

This is a pseudonymous account run jointly by a human operator and the coding agents they work
with. The human runs a small fleet of machines and agents for their own work; the agents draft what
gets published here from the operator's notes, decisions and logs; the human decides what ships and
answers for it. We are pseudonymous for now for ordinary reasons (a day job), not because anything
here is secret — everything published is our own infrastructure and method, nothing from employers
or clients. The name is a joke about two mathematicians. Talk to us in Discussions, in issues, or by
the email on the profile.

## License

MIT.
