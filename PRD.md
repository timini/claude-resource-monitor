# Product Requirements Document: Claude Resource Monitor

Date: 12 October 2026  
Status: Draft for implementation  
Initial platform: macOS, local Claude Code sessions

## 1. Problem

Claude Code can start more subagents and supporting processes than a computer can sustain. Parallel builds, test runners, browsers and servers increase memory use. Worktrees, dependencies, caches and generated artifacts consume disk space. The main agent lacks a dependable, current view of those constraints, so it can continue launching work while the host is under pressure.

The user needs the agent to see the health of its execution environment throughout a session, with enough detail to identify Claude-related consumption and competing applications. Awareness alone is insufficient: configurable admission controls must prevent additional agent launches when capacity is exhausted.

## 2. Goals

- Give the main agent fresh memory, process, agent and disk information during autonomous work.
- Show the same underlying information in a persistent human-readable dashboard.
- Enforce configurable limits on new agent launches independently of model compliance.
- Coordinate limits across multiple local Claude Code sessions.
- Identify resource growth and likely contributors without claiming exact attribution where it is unavailable.
- Keep monitoring inexpensive and independent of additional model calls.

## 3. Non-goals

- Running the Claude model locally or measuring remote model memory.
- Exact per-subagent RAM accounting when agents share a process.
- Automatically terminating unrelated applications, killing existing agents or deleting files.
- Guaranteed containment of all shell-launched processes or all disk writes.
- Remote/cloud monitoring or Linux/Windows support in the first release.
- A full replacement for Activity Monitor or an operating-system resource limiter.

## 4. User experience

When a session starts, the mod registers it with a local collector and displays a compact resource summary. Claude receives a timestamped snapshot before its initial work and refreshed snapshots during the tool loop. A dashboard shows host pressure, Claude consumption, active agents, other large applications and disk capacity.

Before Claude launches another agent, the policy checks the active count, pending reservations, current memory conditions and relevant volume capacity. If a limit is reached, the launch is refused with a factual explanation. Claude can continue work serially or wait for existing tasks to finish. The system does not repeatedly ask the user to approve rejected launches.

Illustrative agent context; these are example measurements, not readings from a real machine:

```text
Host resources — sampled 2026-10-12T13:32:05Z; age 2s
RAM: 32 GiB total; memory pressure HIGH; swap 6 GiB
Claude sessions and descendants: estimated RSS sum 7.4 GiB
This session: 4 active agents; machine limit: 4
Other large apps: Chrome 5.2 GiB; Docker 4.1 GiB
Workspace volume: 11 GiB available; critical threshold: 10 GiB
Tracked workspaces/worktrees: 18 GiB; measured 60s ago
Agent admission: BLOCKED — active-agent limit reached
```

The expanded dashboard includes a complete accessible process list, sortable by memory, and grouped application totals where grouping is available. Agent context contains only the largest consumers and material changes to avoid unnecessary token use. A detail command exposes the full snapshot on demand.

## 5. Functional requirements

### 5.1 Host memory

Report physical RAM, OS memory-pressure state or an explicitly labelled approximation, free/reclaimable memory where available, compressed memory and swap use. Raw free RAM alone must not determine health on macOS. Missing metrics remain unknown rather than becoming zero. Each sample records collection time and freshness.

Show pressure trends over a short rolling window. Swap totals provide context; policy should consider pressure and swap growth rather than treating any nonzero swap as a critical condition.

### 5.2 Processes and attribution

Collect PID, parent PID, process name and resident memory for processes visible to the current user. Track Claude session roots and descendants, including builds, test runners, browsers, MCP servers and other supporting processes where ownership can be established.

Report this session's process group, all tracked Claude sessions, known shared services, and other applications separately. Avoid double counting a PID across sessions. Processes that detach or are shared must be labelled shared or unattributed unless ownership is confirmed.

An RSS sum is an estimate and can count shared memory more than once. Label it accordingly; do not imply it equals unique physical consumption. Shared-process subagents display status and identity, with per-agent RAM marked unavailable. Do not invent agent memory by dividing process memory by agent count.

### 5.3 Agent lifecycle

Track active agents and pending launches using the supported Claude Code lifecycle and agent APIs. Distinguish running, pending, waiting, idle, completed, failed and killed states as supported by the installed version. Make explicit which states count toward the limit; default to counting every live agent that can resume work, excluding the main agent.

Reconcile lifecycle events with current session state. Maintain stable session/agent identifiers and remove stale records after confirmed session termination or lease expiry. A missing completion event must not leak capacity indefinitely.

### 5.4 Disk

Report available and total space on the volumes containing the workspace, tracked worktrees, Claude data and temporary files. Group paths on the same volume so available space is not counted repeatedly. Use filesystem-available bytes and explain that snapshots, purgeable data, quotas and shared APFS storage can affect interpretation.

Measure sizes and growth of explicitly tracked workspaces, worktrees, caches and temporary directories. Directory measurements run less frequently than volume checks, are bounded in duration and do not recursively scan the whole computer. Label overlapping directory totals to prevent double counting. Track recently changed directories when practical, without claiming that every byte was created by Claude.

### 5.5 Context delivery

Provide compact context at session start/resume, user prompt submission and after tool batches, including during long autonomous turns. Use supported hooks such as `additionalContext` rather than relying only on a startup prompt amendment. Provide subagents with relevant resource context too.

The latest snapshot identifies its timestamp, scope and validity. Old snapshots must be treated as historical. Refresh after compaction through a supported lifecycle path. Suppress unchanged detail where possible, but publish a fresh summary when its age limit is reached or policy changes.

Mods can alter system-prompt sections, but exact refresh and caching behavior must be verified on the target build before choosing that path for changing telemetry. Persistent policy instructions and changing measurements should remain separate. The dashboard itself is not model context.

### 5.6 Admission control

Support configurable per-session and machine-wide concurrent-agent limits. Check fresh host conditions before launch; use a coordinated atomic reservation so simultaneous requests cannot each consume the same final slot. Release reservations on failed launches, completion, cancellation and expiry. Account for agents in other registered sessions.

For the initial policy, start with a machine-wide limit of two live subagents, configurable after testing. Memory and disk thresholds must be configurable and documented as provisional. Proposed starting disk thresholds are warning below the greater of 20 GiB or 10% of capacity, and critical below the greater of 10 GiB or 5%. Verify these on representative volumes before release.

On critical memory pressure or critical disk capacity, reject new launches. Use hysteresis and a recovery interval to avoid repeatedly switching between allowed and blocked. The denial includes the sampled state, limiting condition and available alternatives. Existing work continues unless the user explicitly configures a stronger intervention.

When telemetry is missing or stale beyond the configured maximum age, refresh within a short timeout. If that fails, default to refusing new agent launches with a clear monitoring-unavailable reason, while allowing ordinary session work. Expose an explicit user-configured observe-only mode.

The implementation must identify and test the actual agent-launch paths in the target Claude Code build. Count checks alone are not a RAM reservation guarantee: a single admitted agent can launch a large process. If the platform exposes no interceptable launch path, show that enforcement is unavailable rather than claiming protection.

### 5.7 Dashboard and configuration

Show health, memory pressure, swap, Claude process totals, live-agent count/limit, relevant disk availability, sampling age and admission status in a compact band or pane. An expanded view shows processes and tracked directories. Support configuration for sampling intervals, directory roots, limits, warning/critical thresholds, recovery behavior and context verbosity.

Use one shared snapshot for the UI, context and policy. The UI can update between model calls; Claude receives the snapshot on its next request. Collection must continue without driving new model calls.

## 6. Proposed architecture

1. **Local collector:** one lightweight service per user/machine, shared across sessions. Samples OS metrics, maintains process ownership and caches bounded directory measurements.
2. **Session adapter:** a Claude Code plugin/mod registers sessions, observes agent lifecycle, delivers context and renders the dashboard.
3. **Policy coordinator:** atomically evaluates launch requests and maintains reservations and leases across sessions. Use a transaction-capable store or locked local service; do not assume a simple key-value write is an atomic compare-and-reserve operation.
4. **Snapshot schema:** versioned JSON with timestamps, metric validity, host memory, volumes, processes, session/agent states, attribution confidence and policy decisions.

Use current documented mod APIs for process access, timers, UI and agent enumeration where suitable. Classic hooks may provide context injection and launch interception. The feasibility milestone decides the supported combination against actual installed API types and behavior.

## 7. Privacy and operational behavior

Operate locally with no telemetry export or external monitoring backend by default. Summaries injected into Claude context are sent to the model provider as part of normal inference; explain this clearly in setup. Include application names and resource totals, while omitting command arguments, environment variables and sensitive paths by default.

Treat process and file names as untrusted data: sanitize control characters, bound their lengths and render them as factual data. Do not execute strings derived from process names. Store runtime data outside the repository with bounded retention and disk usage. Installing the mod does not authorize automated deletion or process termination.

## 8. Performance targets

Initial targets, to be validated rather than advertised as measured results:

- Host/process sampling every 5 seconds; relevant volume sampling every 10 seconds.
- Bounded directory measurements every 60 seconds or longer.
- Cached admission decisions within 100 ms at the 95th percentile; refresh timeout at most 2 seconds.
- Shared collector resident memory below 50 MiB and average CPU below 1% of one core during steady-state use on the reference machine.
- Routine agent context below 250 tokens, with a hard length bound and controlled update cadence.
- No growing backlog of samples, spawned collectors or directory-scan workers.

## 9. Acceptance criteria

| Scenario | Required result |
| --- | --- |
| Normal session | UI and agent context show consistent metrics, timestamps and policy state. |
| Long autonomous run | Resource context refreshes during the tool loop without another user message. |
| Concurrent launch requests | With one slot left, at most one request obtains a reservation. |
| Multiple sessions | Machine-wide limits include live agents from every registered session. |
| Critical memory or disk | New agent launches are refused with a current, actionable reason. |
| Stale or failed collection | Unknown values remain unknown; launch policy follows the documented failure mode. |
| Failed launch or crashed session | Reservations and leases recover without permanent capacity loss. |
| Shared-process agents | Agent counts are available; unsupported per-agent RAM is explicitly unavailable. |
| Compaction or resume | A fresh snapshot replaces reliance on historical context. |
| Detached/shared processes | Attribution uncertainty is visible and PID totals are not double counted. |
| Disk growth | Relevant volume capacity and tracked-directory growth are visible without unbounded scans. |
| Large process inventory | Full list remains usable; model context remains bounded. |
| Mod disabled | Monitoring stops cleanly and does not leave orphan collectors or change unrelated settings. |

Verify deterministic policy and reservation behavior with simulated metrics and concurrency tests. Run integration checks against actual Claude Code launch paths, lifecycle, compaction and context delivery. Compare host/process and disk readings with OS tools using compatible definitions. Test resource-pressure transitions with injected samples before considering controlled real workloads.

## 10. Delivery milestones

1. **Feasibility:** verify target version, mod types, launch interception, context refresh, agent enumeration and cross-session coordination. Record limits and supported launch paths.
2. **Monitoring MVP:** macOS collector, versioned snapshots, compact dashboard, process details and tracked disk sizes.
3. **Agent awareness:** bounded context injection through autonomous turns, resume and compaction.
4. **Enforcement:** machine-wide reservations, per-session limits, pressure policies, recovery and failure handling.
5. **Release:** integration coverage, measured overhead, installation/configuration guide, compatibility matrix and documented limitations.

## 11. Open implementation questions

- Which installed Claude Code versions expose all required lifecycle and interception paths?
- Which macOS metrics are accessible without elevated permissions, and how closely do they match Activity Monitor?
- How should shared MCP services and detached child processes be attributed?
- What thresholds and concurrency defaults work across 8, 16, 32 and 64 GiB machines?
- Can selected expensive tool operations receive additional admission checks without brittle shell parsing?
- What disk growth usually causes incidents: worktrees, dependencies, build artifacts, session logs or temporary files?

These questions do not prevent the monitoring MVP, but enforcement claims require verified coverage.

## 12. Documentation references

API details are version-sensitive. Recheck these primary sources during implementation:

- [Claude Code mods overview](https://code.claude.com/docs/en/plugins/mods/overview)
- [Mods API and events reference](https://code.claude.com/docs/en/plugins/mods/reference)
- [React to events with a mod](https://code.claude.com/docs/en/plugins/mods/events)
- [Hooks: add context for Claude](https://code.claude.com/docs/en/hooks#add-context-for-claude)
- [Claude Code status line](https://code.claude.com/docs/en/statusline)
- [CLI system-prompt options](https://code.claude.com/docs/en/cli-reference#system-prompt-flags)
