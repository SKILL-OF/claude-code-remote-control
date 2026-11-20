# claude-code-remote-control

Canonical agent procedure for recovering a Claude Code session's per-session `/remote-control` tunnel after an account switch invalidates it. Delegated-recovery pattern — not shared account state.

A session's Remote Control tunnel is per-session and per-account. After a login dance (account switch), every agent needing harness-level visibility (ListAgents, SendMessage) or user visibility on phone/web must have its tunnel refreshed. This cannot be done by the agent itself reliably — an external guardian must do it through the meta-harness.

---

## Why a guardian is required

`/remote-control` presents an interactive menu. An agent running it on its own session risks getting stuck in the menu and losing its turn. The guardian pattern externalizes the interaction: a separate agent (for example, a dispatcher) drives the command on each subject agent's terminal via meta-harness tools (wmux terminal_send, terminal_read).

---

## State Machine — /remote-control Recovery

```mermaid
stateDiagram-v2
    [*] --> IdentifyAgents

    IdentifyAgents --> ForEachAgent : surface_list → collect all agents\nneeding harness visibility

    ForEachAgent --> ReadTerminal : guardian terminal_read on subject's ptyId

    ReadTerminal --> ConfirmActiveAgent : verify subject is live Claude Code session\n(not a bare shell or exited agent)
    ReadTerminal --> Skip : surface is bare shell or terminated agent

    Skip --> NextAgent

    ConfirmActiveAgent --> CheckBuffer : read subject's input buffer
    CheckBuffer --> HasUnsentText : unsent text present in buffer
    CheckBuffer --> BufferClear : buffer is empty

    HasUnsentText --> RecordBuffer : note buffer content in registry\ndo NOT submit or overwrite
    RecordBuffer --> SendCommand

    BufferClear --> SendCommand

    SendCommand : guardian sends /remote-control\nvia terminal_send to subject's ptyId

    SendCommand --> WatchMenu : guardian loops terminal_read

    WatchMenu --> MenuAlreadyConnected : shows "already connected" state\nwith disconnect/reconnect options
    WatchMenu --> MenuNeverConnected : shows initial connection options
    WatchMenu --> MenuPreviouslyConnected : shows "Previously Connected" stale state
    WatchMenu --> TunnelActive : remote-control URL appears in terminal\n(no menu interaction needed)

    MenuAlreadyConnected --> NavigateReconnect : select reconnect option\n(not just disconnect — complete the reconnect)
    MenuNeverConnected --> SelectConnectionOption : choose correct connection option
    MenuPreviouslyConnected --> SelectConnectionOption : treat same as never-connected

    NavigateReconnect --> TunnelActive
    SelectConnectionOption --> TunnelActive

    TunnelActive --> VerifyURL : confirm URL visible in subject's terminal\ncopy URL for user reference if needed
    VerifyURL --> MarkVisible : record agent as harness-visible
    MarkVisible --> NextAgent

    NextAgent --> ForEachAgent : more agents remaining
    NextAgent --> [*] : all agents done

    note right of WatchMenu
        Guardian must NOT abandon subject mid-menu.
        A stuck menu loses the subject's turn.
        Loop until resolved or error confirmed.
    end note

    note right of ConfirmActiveAgent
        Identity check: surface title, agentStatus
        from surface_list. Pane placement alone
        is NOT identity. Ask identity question
        if uncertain before sending /remote-control.
    end note
```

---

## When to run this procedure

Run after every login dance (account switch). All sessions lose harness visibility when the account changes — Remote Control tunnels are per-session and per-account.

Also run when:
- An agent reports it cannot be found via ListAgents
- The operator cannot reach an agent on phone/web
- Remote Control shows "Previously Connected" stale state

---

## Priority order for recovery

Recover in this order:
1. The guardian agent itself (a dispatcher or equivalent) — must be harness-visible to coordinate others
2. The coordinator (a coordinator agent or equivalent) — needs harness visibility to manage subagents
3. Active subagents with ongoing work
4. Idle agents (can wait)

---

## Relation to login dance

This procedure is the final step of every login dance. The full dance state machine is part of the account-switch login procedure, which is not public. The `/remote-control` recovery runs at state `RemoteControlRecovery` in both Flow A and Flow B of that procedure.
