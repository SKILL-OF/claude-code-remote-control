# claude-code-remote-control

Canonical procedure for recovering a Claude Code session's `/remote-control`
tunnel (the human's mobile/web remote-access link to that exact session)
after it gets invalidated — most commonly as a side effect of switching the
`claude` CLI account (see [`SKILL-OF/claude-code-account-login`](https://github.com/SKILL-OF/claude-code-account-login)).

Validated on: Windows 10, `wmux` multi-agent workspace, Claude Code v2.1.266.
The general pattern (below) is not wmux-specific; the concrete commands are.

---

## The key fact that makes everything else follow

`/remote-control` is **per-session**, not shared account state. It binds a
tunnel to the exact terminal process running it. This is structurally
different from `/login`, which is shared across every Claude Code
instance/terminal on the machine:

| Command | Scope | Where it can safely run |
|---|---|---|
| `/login` | shared, machine-wide | **any** terminal/instance — a disposable one, never the session you're protecting |
| `/remote-control` | per-session | **only** the exact session whose tunnel needs fixing — there is no substitute surface |

Because `/remote-control` must run on the exact session that needs
recovering, and because [no interactive command may ever be injected into a
session's own live pty from within its own turn](#why-you-cannot-do-this-yourself)
(self-collision — the target session cannot fix itself), recovering it is
inherently a **delegated** operation: some other, independent execution
context has to drive the target session's terminal from outside.

## Why you cannot do this yourself

A running Claude Code turn that tries to `terminal_send` an interactive
command into its *own* pty is racing its own live output stream against
whatever that command renders next — confirmed, concretely: injecting
`/login` into a session's own terminal while it was still actively
streaming a response produced garbled, unrecoverable output. This is true
even from a same-process Bash-tool subprocess, which still shares the
calling session's process tree and does not provide real isolation.

The only safe pattern is a **second, independent agent/process** — one with
its own turn-cycle — driving the target session's pty from outside.

## The procedure

1. **Pick the delegate.** Prefer a standing agent that has already lived
   through this exact recovery and been corrected on it once, over
   spawning a fresh one. A just-corrected agent carries both the procedure
   *and* its specific failure modes in context; a brand-new spawn carries
   neither, and typically has no tool permissions configured yet — it will
   reliably stall on its own first-use approval prompt instead of doing
   useful work. See "further reading" below.

2. **Give it exactly one complete, unambiguous brief.** Include:
   - The target session's identifier (in wmux: its `ptyId`).
   - The exact menu shape it will see when it sends `/remote-control`:
     three items, **"Disconnect this session" / "Show QR code" /
     "Continue"**, in that order, cursor defaulting to the last item
     ("Continue").
   - The explicit discipline: after every cursor-move keypress, re-read
     the pane and visually confirm the cursor's actual position before
     pressing enter — never assume a keypress landed.
   - The literal done-condition to report back: a line of the form
     `/remote-control is active · ... https://claude.ai/code/session_XXXXXXXX`.
   - An explicit instruction to send nothing else afterward.

3. **Send nothing else, ever, once dispatched.** Only passively read the
   delegate's own output to observe progress. An abort or follow-up
   instruction landing on the target session *while the delegate is
   mid-sequence* is what actually causes real damage — not the delegation
   itself, the **interruption of an in-flight one**. If something looks
   slow or ambiguous, wait; do not intervene.

4. **Handle the "already disconnected" edge case correctly.** If the
   target session's tunnel was already fully down (not merely stale) when
   `/remote-control` is sent, it connects **directly** — a single status
   line, no disconnect menu at all, because there is nothing to
   disconnect first. A delegate correctly briefed to expect a three-item
   menu should treat this mismatch as a signal to **stop and report**
   rather than improvise past it. The correct read of "no menu, tunnel
   already active" in that specific case is **success**, not failure —
   confirm this explicitly if the delegate stops to ask.

5. **Verify through an authoritative side channel**, not just the
   delegate's own narrated success. A genuinely new remote-control session
   URL appearing independently (e.g. in the operator's own current
   context/attribution metadata, separate from anything the delegate
   said) is real confirmation; a delegate's self-report alone is not.

## wmux-specific mechanics

On `wmux`, step 2's delegate drives the target purely through
`mcp__wmux__terminal_read`, `mcp__wmux__terminal_send`, and
`mcp__wmux__terminal_send_key` — never anything that touches its own pty.
Concretely:

```
terminal_read({ptyId: TARGET, tail_lines: 20})               # confirm idle first
terminal_send({ptyId: TARGET, text: "/remote-control", submit: true})
terminal_read({ptyId: TARGET, tail_lines: 15})                # see the menu (or the direct-connect line)
terminal_send_key({ptyId: TARGET, key: "up"})                 # x2, reading + confirming cursor after EACH
terminal_send_key({ptyId: TARGET, key: "enter"})              # only once cursor is confirmed on "Disconnect this session"
terminal_read(...)                                            # confirm "Remote Control disconnected."
terminal_send({ptyId: TARGET, text: "/remote-control", submit: true})
terminal_read(...)                                            # capture the new session URL
```

A known, recurring wmux gotcha independent of this procedure:
`terminal_send({submit: true})`'s bundled `\r` can be silently absorbed by
bracketed-paste instead of committing. If a sent command shows up
queued/pasted but not executed, follow up with a bare
`terminal_send_key({key: "enter"})` and re-verify before assuming the send
failed.

Also prefer checking `pane_list`'s `agentStatus` field over repeated plain
`terminal_read` polling to distinguish "genuinely stuck" from "still
working" — a delegate can look unchanged across several reads while doing
real background work; `agentStatus: "running"` vs `"awaiting_input"` (the
latter usually means it's stalled on its own tool-permission prompt, which
is safe to unstick with a plain `enter` on *its own* pane) disambiguates
this without guessing.

## Further reading

- [`SKILL-OF/claude-code-account-login`](https://github.com/SKILL-OF/claude-code-account-login)
  — the account-switch procedure whose completion is what usually
  necessitates this recovery.
- [`QUESTS-OF/claude-code-remote-control`](https://github.com/QUESTS-OF/claude-code-remote-control)
  — specific real runs of this procedure, including edge cases
  encountered and how they were resolved.
