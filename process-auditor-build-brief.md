# Build Brief: Process Auditor Agent
Source spec: agent-process-playbook.md (fetched 2026-10-13). Builder: any agent with shell access on Jamie's macOS setup. No dependence on external credentials.

## 1. Agent Definition
- **Name**: `ProcessAuditor` (invoked as `auditor`).
- **Role**: Pattern-level watchdog. Reads failure traces across many tasks, classifies them, reports, and proposes process fixes.
- **Scope (hard rule)**: Audits PROCESS, never fixes OUTCOMES. It may say "host X is blocked, switch hosts" or "verification pass was skipped 4 times." It may NOT regenerate the image, resend the file, or edit a deliverable. Outcome work belongs to the worker agent.
- **Authority**: Read-only over worker artifacts. Write access limited to its own reports dir and a fixes ledger file.

## 2. Inputs
- Task logs: `~/agent-logs/tasks/` (or wherever workers append `TASK <id> START/OK/FAIL <detail>` lines).
- Send histories / delivery receipts: parse for HTTP status codes, exit codes, tool error strings.
- Error messages and retry traces: every retry pair (attempt N, error, next attempt).
- Suggested log convention if none exists: one JSONL file per day, `{"ts","agent","task_id","action","status","error","host"}`. Workers append one line per attempt.
- **If Jamie's setup lacks structured logs**: do NOT build a parser against nothing. Step 1 of the build is a shim script (`audit-shim.sh`) that wraps worker invocations: `ts=$(date -u +%FT%TZ); echo "{\"ts\":\"$ts\",\"agent\":\"$1\",...}" >> ~/agent-logs/$(date +%F).jsonl`. Deploy the shim first, run 3 days of traffic, then build classification on real data. Never classify from memory or guesswork.

## 3. Classification Taxonomy + Fix Mapping
| Class | Signals (grep patterns) | Fix (from playbook) |
|---|---|---|
| **delivery** | "failed to download", curl non-200, host names (catbox, 0x0.st, transfer.sh), transport/timeout errors | Host file at public URL first, `curl -I` for HTTP 200 BEFORE handing to tool. Rotate up to 3 hosts, then escalate with a ready-to-run command. Never conclude "impossible" after one host. |
| **auth** | 401/403/404-on-POST, "unauthorized", token/permission errors | Stop retrying immediately — retries don't fix auth. Flag same-day to human; the fix is a credential/config change, not an agent action. |
| **model** | 429/5xx, "rate limit", "context length", empty/garbage output | Backoff + retry once. Log repeat offenders by model/task shape. If same model fails same shape 3x, flag for model or prompt change. |
| **verification** | Task marked OK but no read-back/vision/HTTP-check trace; complaint "output dropped element X" after "done" | Root cause is missing verification pass, not the bug itself. Rule to promote: every deliverable gets independent verification before "done" is reported. A successful tool call is NOT a successful task. |
| **silent** | No ack within N minutes of task assignment; no progress ping on jobs >30min | Process fix: ack on receipt, progress pings on long jobs. Human reads the queue, not the transcript. |
| **burn** | >3 retries on one step, token/time spend per task outliers | 3-strike rule: after 3 failed approaches, hand to human with instructions. Log it, don't let it loop again. |

## 4. Cadence + Scheduling
- **Daily summary** (09:00 local): counts by class, new failure types, unresolved auth flags.
- **Urgent flags**: same-day, emitted immediately when classifier hits auth or burn patterns.
- **Weekly full audit** (Monday 08:00): everything below in §5.
- cron (macOS crontab, or launchd if preferred):
```
# daily summary
0 9 * * * /Users/jamie/agents/auditor/daily_summary.sh >> /Users/jamie/agents/auditor/logs/cron.log 2>&1
# weekly full audit
0 8 * * 1 /Users/jamie/agents/auditor/weekly_audit.sh >> /Users/jamie/agents/auditor/logs/cron.log 2>&1
```
- Daily script should also run a registry diff (playbook §6): workers/names/status live in a file, not memory.

## 5. Weekly Report Template
```
# Process Audit — week of YYYY-MM-DD
## Failure counts by class
delivery: N | auth: N | model: N | verification: N | silent: N | burn: N
## Fixes applied (with task IDs)
- <class>: <what changed> (<task ids>)
## Fixes to PROMOTE into skills/playbook
- <imperative rule> + why  (playbook rule: write it down the moment it's learned)
## Repeat offenders
- <host/tool/model>: N failures, trend up/down
## Open escalations
- <item>: waiting on human since <date>
```

## 6. Escalation Rules
- **3-strike rule** (from playbook): same task/step fails via 3 distinct approaches → escalate to human with a ready-to-run command or explicit instruction. Escalation is not failure.
- Immediate escalation (skip strikes): auth failures, anything irreversible risked, same class failing >5x/day (systemic, not task-level).
- Every escalation lands in a single inbox file (e.g. `~/agents/auditor/escalations.md`) — never buried in chat scrollback.

## 7. Day-One Smoke Test
1. Create fixture log with 5 planted entries: 1 delivery (curl 000), 1 auth (401), 1 model (429), 1 verification (OK with no read-back trace), 1 silent (no ack).
2. Run classifier: expect exactly 5 correct labels, correct counts.
3. Run daily_summary.sh: expect report file with the same counts.
4. Run escalation path on the auth entry: expect line appended to escalations.md.
5. Run weekly_audit.sh: expect template filled with fixture numbers.
6. Pass = all 5 outputs match fixtures. Any mismatch = the auditor itself is a delivery-class bug; fix before deploying.

## 8. Adversarial Review — what's wrong or fragile in the playbook
1. **No log spec.** The auditor spec lists "task logs" as input but the playbook defines no log format anywhere. Classifying free-text chat is unreliable. Mitigation: the JSONL shim in §2 is mandatory, phase one of the build.
2. **"3 hosts then escalate" can be a loop.** 3-strike applies per task, but nothing caps total retries across a day. Add the burn class and daily aggregate cap (>5 same-class/day = systemic flag) — done in §6.
3. **Registry-diff-daily is stated, not operationalized.** No file format, no diff threshold. This brief defers it; builder should define a registry file format before automating the diff, or it will produce noise.
4. **"Verification pass" is undefined for non-image outputs.** Vision pass for images is clear; "read-back for data" needs a concrete check per output type (HTTP status for sends, field-diff for data files). Builder must enumerate checks per deliverable type or agents will fake it.
5. **Host list will rot.** catbox/0x0.st/transfer.sh statuses change monthly. Treat host status as data (a checked-in `hosts.yaml` re-probed weekly by the auditor), not as hardcoded playbook truth.
6. **Unmeasured principle**: "most failures are delivery" is asserted, not counted. The auditor's first weekly report should confirm or refute the base rate — if delivery is <30% of failures, reweight the taxonomy priorities.
