---
name: "Cohesity-Protection-Coverage-Audit"
description: "Build the \"Cohesity Protection Coverage & Policy Compliance Audit\" from live Helios data via the helios-mcp connector: protected vs unprotected objects by environment and cluster, protection policies checked against frequency / retention / offsite / immutability standards, paused or SLA-violating or never-run protection groups, protected objects whose last run failed, stalled replication and archival, source registration health and snapshots past retention, with ranked findings. Publishes it as a live claude.ai Artifact and refreshes it every 4 hours. Also answers narrow questions in chat (objects with no recent good backup in a window, or the retention and replication state of one protection group). Use when the user asks for a protection coverage report, unprotected or at-risk objects, policy compliance or retention audit, replication problems, backup gaps, an audit of Helios protection posture, or to refresh/schedule that audit."
---

# Cohesity Protection Coverage & Policy Compliance Audit

Builds a single-page audit from live data using the **helios-mcp** connector tools, publishes it as a persistent **Artifact** (private page with a stable URL), and refreshes that same artifact **every 4 hours** via a scheduled task. The full page template is embedded at the end of this skill, so no other file is needed. Never invent numbers. If a tool fails or returns nothing, show "n/a" and say so in the final message. Never write a loose static HTML file as the deliverable. Everything is read-only.

## What the page contains

Header with eyebrow "COHESITY PROTECTION AUDIT", title "Protection Coverage & Policy Compliance", freshness line and "Refreshes every 4 hours". A **cluster multi-select** (all registered clusters, tagged connected / not connected) recalculates, in the browser, the KPIs, coverage-by-cluster, policy table and exceptions from the embedded data (it does not query Helios live) and is remembered in localStorage.

- KPI cards: Protection coverage % (protected / (protected + unprotected)), Unprotected objects, Policy compliance (policies passing), Exceptions (paused + SLA-violated + never-run groups).
- **Findings** (ranked high / medium / low, written by you in step 8).
- Panels: Coverage by environment (stacked bars, Helios-wide), Coverage by cluster (bars of % protected), Policy compliance split (pass / warn / fail).
- Tables: Policy compliance detail (frequency, retention, offsite copy, DataLock, groups using it, status, issues); Group exceptions (paused with age, SLA violated, never run, last run failed, replication stalled, archive failed). The Unprotected KPI also shows how many protected objects had a failed last run.
- Text cards (Helios-wide): Source registration health, Snapshots past policy retention.
- Footnote: the standards used, the replica double-counting caveat and any scope limits.

The template (see "Page template") already implements all of this. Do not redesign it; fill the placeholders.

## Standards (defaults, overridable by the user)

`standards = {"maxFreqHours": 24, "minRetentionDays": 30}`. These are the skill's baseline, **not** Cohesity or regulatory policy; if the user gives their own (e.g. "RPO 4 h, 90 days"), use those and say so in the scope note. Policy status rules:

- **fail**: backup frequency interval is longer than `maxFreqHours` (ProtectOnce policies are exempt from the frequency rule), or the longest retention (base or extended) is shorter than `minRetentionDays`.
- **warn**: no offsite copy (no replication target and no archival/cloud target), or no DataLock (WORM) on base retention.
- **pass**: none of the above.

## Data steps

Compute `now` in **epoch microseconds** (`date -u +%s` x 1,000,000; all date filters and `{{GENERATED_UTC}}` are UTC). Run independent calls in parallel. Large responses are saved to a file: process them with `python3 -I` / jq rather than reading them.

1. **Clusters**: `dataprotect-get-clusters`. Keep id, incarnation id, name, `isConnectedToHelios`. Only connected clusters can be queried with `accessClusterId`. Count ALL entries for the registered total.
2. **Coverage counts**: `dataprotect-search-objects` returns a top-level `count` (total matches) even with `count: 1`, so never page through objects just to count them. Always pass `isDeleted: false` and `includeTenants: true`.
   - Per connected cluster: two calls with `clusterIdentifiers: ["<clusterId>:<incarnationId>"]`, `isProtected: true` and `isProtected: false`, `count: 1`. Put the totals in that cluster's `protected` / `unprotected`.
   - Per environment (Helios-wide): two calls per environment with `environments: ["kVMware"]` etc. Use the environments seen in step 3 plus these common ones: kVMware, kPhysical, kSQL, kOracle, kView, kO365, kAWS, kAzure, kAcropolis, kKubernetes, kHyperV. Drop environments with zero objects, keep at most 10. Strip the leading `k` for display.
   - **Protected but at risk**: per connected cluster, one more call with `isProtected: true`, `lastRunStatusList: ["Failed"]`, `clusterIdentifiers`, `count: 1`; the top-level `count` is that cluster's `atRisk` (protected objects whose last protection run failed). This is the cheap, page-level answer to "which protected objects have no good backup"; the per-object version is the chat drill-down below.
   - Sample up to 10 unprotected objects (`isProtected: false`, `count: 10`) for the findings text; do not list object-level data in the page.
3. **Protection groups**: `dataprotect-get-protection-groups` per connected cluster with `accessClusterId`, `includeLastRunInfo: true`, `isActive: true`, `isDeleted: false`, `useCachedData: true`, `pruneSourceIds: true`, `pruneExcludedSourceIds: true`. A cluster call can time out: retry once, then mark that cluster `na`. Keep id, name, `environment`, `isPaused`, `pauseMetadata.lastPauseModificationTimeUsecs`, `policyId`, `lastRun.localBackupInfo` (status, isSlaViolated, endTimeUsecs). Build `exceptions` with **one row per group** carrying a `types` array and a combined `detail`, so the Exceptions KPI counts groups, not rows:
   - `paused`: `isPaused` true; detail "paused N d" from `pauseMetadata.lastPauseModificationTimeUsecs`. Some paused groups have no `pauseMetadata`: fall back to `lastModifiedTimestampUsecs` and write "paused (age from last edit) N d".
   - `sla`: last run `isSlaViolated` true.
   - `neverrun`: not paused and no `lastRun`.
   - `failed`: last run status `Failed`.
   - `repl`: the last run's `replicationInfo.replicationTargetResults[]` has a result with status `Failed`, or the group's policy has a replication target but the last run has no replication result at all. Detail: "replication to <target cluster> failed". When replication is configured and the latest result failed, say "likely cause: replication stopped".
   - `arch`: the last run's `archivalInfo.archivalTargetResults[]` has a result with status `Failed` (ignore `Accepted` and `Skipped`). Detail: "archive to <target name> failed".
   Example: `{"group":"M365-User","cluster":"...","env":"O365OneDrive","types":["sla","failed"],"detail":"Token refresh failed"}`. Sanitise `detail` (no email addresses, no GUIDs).
4. **Policies**: `dataprotect-get-protection-policies` per connected cluster (omit `includeStats`, it can error). Units come back mixed (Days, Weeks, Months, Years) regardless of `durationUnit`, so normalise yourself: Weeks x 7, Months x 30, Years x 365; frequency to hours (round to one decimal, so a 10-minute policy is 0.2): Minutes/60, Hours x1, Days x24, Weeks x168, Months x720; a `ProtectOnce` schedule sets `protectOnce: true` and `freqH: null`. For each policy record `retDays` (base `backupPolicy.regular.retention`), `extDays` (the longest of `extendedRetention` and of any replication or archival target retention in `remoteTargetPolicy`, else 0; the retention rule below uses `max(retDays, extDays)`, so a 28-day local policy with a 1-month archive copy passes a 30-day standard), `offsite` (a replication target, an archival target, a cloud vault / `rpaasTargets` entry in `remoteTargetPolicy`, or a primary backup target of type `Archival`), `lock` (base retention has `dataLockConfig`), `groups` (number of step-3 groups with that `policyId`, counted across all clusters). List only policies used by at least one group when the total number of policies across all clusters exceeds 30; otherwise list all. Apply the status rules above and write short `issues` strings such as "Runs every 48 h (limit 24 h)".
5. **Source registrations**: `dataprotect-get-source-registrations` per connected cluster (`useCachedData: true`). The same registration can be returned more than once (five duplicates were seen on one cluster): dedupe by `id` first. A registration is unhealthy when `authenticationStatus` is not `Finished` or a refresh error message is present. Report it as **stale** only when `lastRefreshedTimeMsecs` is older than 24 h for auto-refreshing sources (VMware, AWS, Azure, Acropolis, O365, Kubernetes) and older than 7 days for NAS, UDA and physical sources, which rarely refresh; applying 24 h to everything marks most of them stale and is noise. Summarise for `{{SOURCE_TEXT}}`: registrations per cluster, unhealthy ones by name and reason. If the call fails, say "n/a: not assessed".
6. **Snapshots past retention** (sampled, optional): for up to 10 groups from the step-4 policies whose retention is shortest, find an object id via `dataprotect-get-protection-group-runs` (`numRuns: 1`, `includeObjectDetails: true`, the group's `accessClusterId`) and call `dataprotect-get-object-snapshots` with `id` and `protectionGroupIds`. Flag snapshots older than the policy's longest retention. Report how many groups were sampled. If not done, write "n/a: not assessed" and why. Put the paragraph in `{{SNAPSHOT_TEXT}}`.
7. **Cluster scope**: connected clusters get real numbers. Clusters that are not connected to Helios get zeros and `na: "not connected to Helios"` (the page hides their coverage bars). If a connected cluster answered the object-count calls but its groups, policies or registrations timed out (retry with `maxResultCount: 25` and without `includeLastRunInfo` first), keep its counts, leave `na` unset, set `partial: "groups, policies and registrations timed out"`, and say so in the scope note; the page shows partial clusters with that note.
8. **Findings**: write 3 to 8 findings, each `{sev, title, detail}` with `sev` one of `high`, `medium`, `low`. Typical: large unprotected populations (give counts and the environment), policies failing the standards, groups paused for a long time, SLA violations, unhealthy source registrations, clusters that could not be queried. Only state what the data supports.

## Filling the template

| Placeholder | Value |
|---|---|
| `{{GENERATED_UTC}}` | e.g. `2026-10-13 21:33 UTC` |
| `{{DATA_JSON}}` | the JSON below, valid JSON, with every `</` written as `<\/` |
| `{{SOURCE_TEXT}}` | one paragraph (step 5), plain text, HTML-escaped |
| `{{SNAPSHOT_TEXT}}` | one paragraph (step 6), may contain `<code>` tags |
| `{{SCOPE_NOTE}}` | footnote starting with `* `: standards used, clusters excluded and why (real registered count), the replica double-counting caveat |

`{{DATA_JSON}}` shape:

```
{"standards":{"maxFreqHours":24,"minRetentionDays":30},
 "clusters":[{"name":"...","connected":true,"na":"optional: cluster could not be queried at all","partial":"optional: counts present, other queries failed","protected":210,"unprotected":87,"atRisk":14}, ...ALL registered clusters...],
 "envs":[{"env":"VMware","protected":410,"unprotected":997}, ...],
 "policies":[{"name":"Gold","cluster":"...","freqH":4,"protectOnce":false,"retDays":7,"extDays":1095,"offsite":true,"lock":true,"groups":5,
   "status":"pass","issues":[]}, ...],
 "exceptions":[{"group":"...","cluster":"...","env":"AWS","types":["paused"],"detail":"paused 34 d"}, ...ONE row per group...],
 "findings":[{"sev":"high","title":"...","detail":"..."}, ...]}
```

Numbers must tie: for any selection the coverage-by-cluster bars use the selected clusters' `protected` / `unprotected`, and the KPI is their sum. Object counts include replicas of the same object on other clusters, so the sum over clusters is larger than the sum over environments (which is Helios-wide and de-duplicated); the scope note must say so.

## Large Helios results (do this in the main session)

A Helios tool result that is too large to return inline (for example `dataprotect-search-objects` or `dataprotect-get-object-snapshots`) is saved to a file such as `/var/folders/.../projects/session/<sessionId>/tool-results/<file>.txt`. That host path is not readable from the Linux shell or from subagents, so a delegated run cannot process the data. Collect and process the data yourself, in the main session:

1. Translate a saved path for bash: replace the prefix up to `/projects/` with `/sessions/<session>/mnt/.claude/projects/` (find the file with `find /sessions -name '<file>.txt'`). Scheduled-task sessions have their own session id.
2. Process the files with python and write one row per cluster or group to a JSON file as you go, so a long run survives a context reset.
3. A result that is small enough to come back inline cannot be read from bash. Compute its figures directly and add them to the same JSON.

## Manual refresh

The template has a **Refresh from Helios** button. A published page cannot call the Helios connector itself, so the second button copies the command `Run the Cohesity-Protection-Coverage-Audit skill to refresh this report from live Helios data`; paste it into Claude and it republishes the same artifact. Keep the button in every rebuild.

## When Helios is unavailable

Make `dataprotect-get-clusters` the first Helios call. If it fails ("Server helios-mcp unavailable", an expired token, or a timeout twice in a row), **stop collecting data and do not clear, blank or overwrite anything the page already holds**. Instead show an error on the existing page:

1. Find the live artifact (its URL is in the scheduled task prompt, or use `Artifact` `action: "list"` and match the title "Cohesity Protection Coverage And Policy Compliance Audit" case-insensitively) and read it with `action: "read"`, `path: "index.html"`, `out_dir` set to the scratch directory. If no artifact exists yet, publish nothing and tell the user.
2. Build an HTML **fragment** from the saved page: everything from the first `<title>` up to (not including) `</body>`. Leave the data JSON, scripts and styles exactly as they are.
3. Add or replace this banner directly after `</header>` (replace the `<!--HELIOS_ERR-->` marker, or an existing `<div class="errbar"` block if one is already there; add the `.errbar` rule to the `<style>` block if the page lacks it):
   `<div class="errbar" role="alert">Unable to connect to Helios. Showing the last successful data. Last attempt <UTC time>. Reconnect the helios-mcp connector, then refresh.</div>`
4. Publish the fragment with `url` set to the artifact, `no icon` (the stored declaration is kept), so the page keeps showing its last data plus the error message.
5. Reply in one line: "Unable to connect to Helios; kept the last data and added an error banner."

A later successful run rebuilds the page from the template, which carries an empty `<!--HELIOS_ERR-->` marker, so the banner disappears by itself. Never invent or reuse old numbers as new ones: the banner is the only change on a failed run.

## Output: publish as a live Artifact

1. Write the filled template to a scratch file as an HTML **fragment** (no doctype/html/head/body) in the session's working/outputs directory (for example `<outputs>/_build/audit.html`) or any scratch folder the Write tool can reach, never inside a git repository; keep `<title>Cohesity Protection Coverage And Policy Compliance Audit</title>` stable. If the `artifact-design` skill is available, load it first.
2. Check whether it already exists: `Artifact` with `action: "list"`, title "Cohesity Protection Coverage And Policy Compliance Audit".
   - **Exists** (any refresh): `action: "read"` it, then publish the new file with `url` set to it. Never create a second artifact, do not pass `icon`.
   - **Does not exist**: publish with `icon: "shield"` and a one-sentence `description`.
   - **Stored URL not found** (the artifact was deleted or is not visible to this account): do not stop. Publish the page as a new artifact (first-publish rules above) and tell the user in the reply: "I couldn't find the previous artifact, so I republished a new one", with the new link, and that any scheduled task prompt still pointing at the old URL needs the new one.
3. The Artifact is the deliverable. Do not also save a static HTML file.
4. Reply in 2 to 3 sentences: coverage %, count of failing policies, the top finding. State any tile that fell back to "n/a". On a scheduled refresh, one line.

## Refresh every 4 hours (default)

After the first publish, call `mcp__scheduled-tasks__list_scheduled_tasks` and look for a task whose title contains "Protection coverage audit". If missing, create `cohesity-protection-coverage-refresh` with title "Cohesity Protection coverage audit (every 4 hours)", `cronExpression: "0 */4 * * *"` (local time), `notifyOnCompletion: false`, and this self-contained prompt: "Run the Cohesity-Protection-Coverage-Audit skill end to end with live helios-mcp data (read-only) and republish the existing 'Cohesity Protection Coverage And Policy Compliance Audit' artifact: find it with the Artifact list action by title (case-insensitive) or use its URL if one is given here (add it when the task is created: ARTIFACT_URL), read it, then publish the refreshed page with its url. Never create a second artifact and do not pass an icon. Reply in one line with coverage %, unprotected objects and failing policies. If helios-mcp is unavailable, follow the skill's \"When Helios is unavailable\" section (keep the last data and add the error banner). If the skill is not installed in Claude, say so and stop; install it from github.com/Cohesity-Maestro/Cohesity-Protection-Coverage-Audit." If a task exists with another cadence, fix it with `update_scheduled_task`. Tell the user scheduled tasks run only while the Claude app is open, a missed run executes on next launch, and the task runs in Auto-approve mode unless they edit it. Keep the `Refreshes every 4 hours` header text in sync with the cron.

## Drill-down modes (chat answers, no republish)

These replace the standalone `helios-coverage-check` and `helios-retention-audit` skills. Use them when the user asks a narrow question; do not republish the artifact unless asked.

**A. Objects with no recent good backup.** Ask for the scope if it is missing: object type or environment, cluster or name filter, and the window (default 24 hours). Then: `dataprotect-get-clusters` for names; `dataprotect-search-objects` for the scope (`isProtected: true`, `count` 50 or fewer, page through the rest with `paginationCookie`); for each object `dataprotect-get-object-snapshots` with the object `id` and the object's `accessClusterId`, keeping the newest successful snapshot; keep objects whose newest successful snapshot is older than the window or missing; `dataprotect-get-protection-groups` to name the group that should protect each one. Answer with the scope, the window and how many objects were checked, then a table (object, cluster, protection group, hours since last good snapshot, sorted by hours descending) and a separate table for objects with no protection group. Say so if a page limit cut the scope short. Do not recommend changes to protection groups; point to Helios.

**B. Retention and replication of one protection group.** Ask for the group name. `dataprotect-get-protection-groups` to find it (keep id, cluster, `policyId`); `dataprotect-get-protection-policies` for the schedule, longest retention (normalise units) and replication targets, quoting values from the policy and never assuming defaults; `dataprotect-get-protection-group-runs` with `numRuns: 10` and `includeObjectDetails: true`, showing the backup result and the replication result of each run and flagging runs that stopped or have no replication result; up to 5 protected objects from the run details, each with `dataprotect-get-object-snapshots` (`id`, `protectionGroupIds`), listing snapshots older than the policy retention. Summarise in four parts: policy retention, the last 10 runs with backup and replication results, snapshots past expiry (count per object, and how many objects you sampled), and the next place to check in Helios. If replication is configured and the last replication run failed or is old, say "likely cause: replication stopped". Do not recommend deleting snapshots: deletion is a Helios action that needs a person's approval.

## Rules
- Read-only against Helios: only get/list/search tools.
- Do not list individual object names, credentials or internal IDs in the page; names of clusters, groups and policies are fine.
- State the standards used. Never present the default thresholds as a regulatory requirement.
- The artifact stays private; tell the user to share it from the page's Share menu.

## Page template

```html
<title>Cohesity Protection Coverage And Policy Compliance Audit</title>
<style>
/* Layout: cluster filter, KPI row, ranked findings, three panels, policy table, exceptions table, two text cards. Audit page, scanned top-down. */
:root{--bg:#f3f6f5;--card:#ffffff;--border:#dbe3e0;--text:#1f2a28;--muted:#5b6a67;--track:#eef2f1;--rule:#eef1f0;--hover:#eef5ef;
--green:#2e7d32;--red:#c0392b;--amber:#d4820a;--green-lt:#81b57e;--red-lt:#e59a93;--amber-lt:#e8bb74;--chip:#e8efec}
@media (prefers-color-scheme: dark){:root:not([data-theme="light"]){--bg:#121917;--card:#1b2422;--border:#2d3a37;--text:#e6eeec;--muted:#98a9a5;--track:#25312e;--rule:#26322f;--hover:#243230;
--green:#6fbf73;--red:#ef7a6c;--amber:#f0a840;--green-lt:#3d6a3f;--red-lt:#7a3c36;--amber-lt:#7a5a22;--chip:#2a3835;color-scheme:dark}}
:root[data-theme="dark"]{--bg:#121917;--card:#1b2422;--border:#2d3a37;--text:#e6eeec;--muted:#98a9a5;--track:#25312e;--rule:#26322f;--hover:#243230;
--green:#6fbf73;--red:#ef7a6c;--amber:#f0a840;--green-lt:#3d6a3f;--red-lt:#7a3c36;--amber-lt:#7a5a22;--chip:#2a3835;color-scheme:dark}
body{background:var(--bg);color:var(--text);font-family:Arial,Helvetica,sans-serif;padding-inline:16px}
*{box-sizing:border-box}
.wrap{max-width:1180px;margin:0 auto;padding-block:28px 40px}
header{display:flex;justify-content:space-between;align-items:flex-end;gap:16px;flex-wrap:wrap;padding-bottom:14px;border-bottom:3px solid var(--green);margin-bottom:16px}
.eyebrow{color:var(--green);font-size:12px;font-weight:700;letter-spacing:.12em}
h1{margin:4px 0 0;font-size:28px;text-wrap:balance}
.meta{text-align:right;font-size:12px;color:var(--muted);line-height:1.6}
.filter{display:flex;align-items:center;gap:12px;flex-wrap:wrap;margin-bottom:16px}
.filter .lbl{font-size:12px;color:var(--muted);text-transform:uppercase;letter-spacing:.05em}
details.ms{position:relative}
details.ms>summary{list-style:none;cursor:pointer;background:var(--card);border:1px solid var(--border);border-radius:8px;padding:8px 12px;font-size:13px;min-width:260px;max-width:100%;display:flex;justify-content:space-between;gap:12px}
details.ms>summary::-webkit-details-marker{display:none}
details.ms>summary::after{content:"\25BE";color:var(--muted)}
details.ms>summary:focus-visible,.menu button:focus-visible,.menu input:focus-visible{outline:2px solid var(--green);outline-offset:2px}
.menu{position:absolute;z-index:5;top:calc(100% + 4px);left:0;width:340px;max-width:calc(100vw - 32px);max-height:340px;overflow-y:auto;background:var(--card);border:1px solid var(--border);border-radius:8px;padding:6px;box-shadow:0 6px 20px rgba(0,0,0,.18)}
.menu .acts{display:flex;gap:8px;padding:4px 6px 8px;border-bottom:1px solid var(--rule);margin-bottom:4px}
.menu button{background:none;border:1px solid var(--border);color:var(--text);border-radius:6px;padding:4px 10px;font-size:12px;cursor:pointer}
.menu label{display:flex;align-items:center;gap:8px;padding:6px;border-radius:6px;font-size:13px;cursor:pointer}
.menu label:hover{background:var(--hover)}
.menu .tag{margin-left:auto;font-size:11px;color:var(--muted)}
.grid{display:grid;gap:16px;margin-bottom:16px}
.g4{grid-template-columns:repeat(4,minmax(0,1fr))}.g3{grid-template-columns:repeat(3,minmax(0,1fr))}.g2{grid-template-columns:repeat(2,minmax(0,1fr))}
@media(max-width:900px){.g4,.g3,.g2{grid-template-columns:repeat(2,minmax(0,1fr))}}
@media(max-width:600px){.g4,.g3,.g2{grid-template-columns:minmax(0,1fr)}}
.card{background:var(--card);border:1px solid var(--border);border-radius:10px;padding:16px 18px;min-width:0;margin-bottom:0}
.full{margin-bottom:16px}
.card h2{margin:0 0 12px;font-size:14px}
.lbl{font-size:12px;color:var(--muted);text-transform:uppercase;letter-spacing:.05em}
.big{font-size:40px;font-weight:700;margin:6px 0 2px;font-variant-numeric:tabular-nums}
.sub{font-size:12px;color:var(--muted)}
.tool{margin-top:10px;font:11px "Courier New",monospace;color:var(--muted);border-top:1px solid var(--rule);padding-top:8px}
.red{color:var(--red)}.amber{color:var(--amber)}.ok{color:var(--green)}
.bar{display:flex;align-items:center;gap:8px;margin:8px 0;font-size:12px}
.bar .n{width:110px;flex:none;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.bar .t{flex:1;background:var(--track);border-radius:4px;height:16px;display:flex;overflow:hidden}
.bar .f{height:16px}
.bar .c{width:78px;text-align:right;font-weight:700;font-variant-numeric:tabular-nums;flex:none}
.legend{display:flex;gap:14px;font-size:11px;color:var(--muted);margin-bottom:6px}
.legend i{display:inline-block;width:10px;height:10px;border-radius:2px;margin-right:4px;vertical-align:-1px}
.tw{overflow-x:auto}
table{width:100%;border-collapse:collapse;font-size:12px}
th{text-align:left;color:var(--muted);font-weight:600;padding:4px 6px;border-bottom:1px solid var(--border)}
td{padding:7px 6px;border-bottom:1px solid var(--rule);vertical-align:top}
td.num{font-variant-numeric:tabular-nums;white-space:nowrap}
td.iss{color:var(--muted);overflow-wrap:anywhere;max-width:300px}
.chip{display:inline-block;border:1px solid var(--border);background:var(--chip);border-radius:999px;padding:2px 8px;font-size:11px;white-space:nowrap}
.chip.pass{border-color:var(--green);color:var(--green)}.chip.warn{border-color:var(--amber);color:var(--amber)}.chip.fail,.chip.high{border-color:var(--red);color:var(--red)}
.chip.medium{border-color:var(--amber);color:var(--amber)}.chip.low{color:var(--muted)}
td .chip{margin:0 2px 2px 0}
.fbar{display:flex;flex-wrap:wrap;gap:8px 12px;align-items:flex-end;margin:0 0 10px}
.fbar label{font-size:11px;letter-spacing:.06em;text-transform:uppercase;color:var(--muted);display:flex;flex-direction:column;gap:4px}
.fbar select,.fbar input{background:var(--card);color:var(--text);border:1px solid var(--border);border-radius:6px;padding:6px 8px;font:inherit;font-size:12px;min-width:0;max-width:100%}
.fbar input{width:170px}
.fbar select:focus-visible,.fbar input:focus-visible{outline:2px solid var(--green);outline-offset:2px}
.tw.sc{max-height:440px;overflow:auto}
.tw.sc thead th{position:sticky;top:0;background:var(--card);z-index:1}
.find{display:grid;grid-template-columns:70px 1fr;gap:10px;padding:8px 0;border-bottom:1px solid var(--rule);font-size:13px;line-height:1.5}
.find:last-child{border-bottom:0}
.find b{display:block}.find span.d{color:var(--muted);font-size:12px}
p{font-size:13px;line-height:1.55;margin:0}
.foot{font-size:11px;color:var(--muted);margin-top:8px;line-height:1.6}
.empty{font-size:12px;color:var(--muted);padding:8px 0}
code{font-family:"Courier New",monospace}
.errbar{background:var(--card,#fff);color:var(--red,#c0392b);border:1px solid var(--red,#c0392b);border-left:4px solid var(--red,#c0392b);border-radius:8px;padding:10px 14px;margin:0 0 16px;font-size:13px;line-height:1.5}
.rfr{display:flex;justify-content:flex-end;align-items:center;gap:8px;flex-wrap:wrap;margin:0 0 12px;font-size:12px;color:var(--muted,#5b6a67)}.rfr button{background:var(--card,#fff);color:var(--text,#1f2a28);border:1px solid var(--border,#dbe3e0);border-radius:6px;padding:6px 12px;font-size:12px;cursor:pointer}.rfr button:focus-visible{outline:2px solid var(--green,#2e7d32);outline-offset:2px}.rfr input{font:12px "Courier New",monospace;background:var(--card,#fff);color:var(--text,#1f2a28);border:1px solid var(--border,#dbe3e0);border-radius:6px;padding:5px 8px;width:300px;max-width:100%}
</style>
<div class="wrap">
<header><div><div class="eyebrow">COHESITY PROTECTION AUDIT</div><h1>Protection Coverage &amp; Policy Compliance</h1></div>
<div class="meta">Source: Helios MCP, read-only<br>Live data, generated {{GENERATED_UTC}}<br>Refreshes every 4 hours</div></header>
<!--HELIOS_ERR-->
<div class="rfr"><span id="rfn">Auto-refreshes every 4 hours.</span><button type="button" id="rf" title="Copy the command that pulls live data from Helios">Refresh from Helios</button></div>
<script>
(function(){
var CMD='Run the Cohesity-Protection-Coverage-Audit skill to refresh this report from live Helios data';
var n=document.getElementById('rfn');
document.getElementById('rf').addEventListener('click',function(){
 function done(ok){if(ok){n.textContent='Copied. Paste it into Claude to pull live data; reopen or refresh this tab when it finishes.'}else{n.textContent='';var i=document.createElement('input');i.readOnly=true;i.value=CMD;i.setAttribute('aria-label','Refresh command');n.appendChild(document.createTextNode('Copy this into Claude; reopen or refresh this tab when it finishes: '));n.appendChild(i);i.focus();i.select()}}
 try{navigator.clipboard.writeText(CMD).then(function(){done(true)},function(){done(false)})}catch(e){done(false)}
});
})();
</script>

<div class="filter"><span class="lbl">Clusters</span>
<details class="ms" id="ms"><summary id="mssum">All clusters</summary>
<div class="menu"><div class="acts"><button type="button" id="selall">Select all</button><button type="button" id="selnone">Clear</button><button type="button" id="selconn">Connected only</button></div><div id="opts"></div></div></details>
<span class="sub" id="selnote"></span></div>

<div class="grid g4">
<div class="card"><div class="lbl">Protection coverage</div><div class="big" id="k1">-</div><div class="sub" id="k1s"></div><div class="tool">dataprotect-search-objects</div></div>
<div class="card"><div class="lbl">Unprotected objects</div><div class="big red" id="k2">-</div><div class="sub" id="k2s"></div><div class="tool">dataprotect-search-objects</div></div>
<div class="card"><div class="lbl">Policies passing</div><div class="big" id="k3">-</div><div class="sub" id="k3s"></div><div class="tool">dataprotect-get-protection-policies</div></div>
<div class="card"><div class="lbl">Group exceptions</div><div class="big amber" id="k4">-</div><div class="sub" id="k4s"></div><div class="tool">dataprotect-get-protection-groups</div></div>
</div>

<div class="card full"><h2>Findings, Ranked</h2>
<div class="fbar"><label for="fs">Severity<select id="fs"><option value="">All</option><option value="high">High</option><option value="medium">Medium</option><option value="low">Low</option></select></label></div>
<div id="fd"></div></div>

<div class="grid g3">
<div class="card"><h2>Coverage By Environment</h2><div class="legend"><span><i style="background:var(--green-lt)"></i>Protected</span><span><i style="background:var(--red-lt)"></i>Unprotected</span></div><div id="p1"></div><div class="foot">Helios-wide, not filtered by cluster.</div></div>
<div class="card"><h2>Coverage By Cluster</h2><div class="legend"><span><i style="background:var(--green-lt)"></i>Protected</span><span><i style="background:var(--red-lt)"></i>Unprotected</span></div><div id="p2"></div><div class="foot" id="p2f"></div></div>
<div class="card"><h2>Policy Compliance</h2><div id="p3"></div><div class="foot" id="p3f"></div></div>
</div>

<div class="card full"><h2>Policy Compliance Detail</h2>
<div class="fbar"><label for="ps">Status<select id="ps"><option value="">All</option><option value="fail">Fail</option><option value="warn">Warn</option><option value="pass">Pass</option></select></label><label for="po">Offsite copy<select id="po"><option value="">Any</option><option value="no">Missing</option><option value="yes">Present</option></select></label><label for="pl">DataLock<select id="pl"><option value="">Any</option><option value="no">Missing</option><option value="yes">Present</option></select></label><label for="pq">Search<input type="search" id="pq" placeholder="Policy name, issue…"></label></div>
<div class="tw sc"><table><thead><tr><th>Policy</th><th>Cluster</th><th>Frequency</th><th>Retention</th><th>Offsite</th><th>DataLock</th><th>Groups</th><th>Status</th><th>Issues</th></tr></thead><tbody id="pt"></tbody></table></div>
<div class="foot" id="ptf"></div></div>

<div class="card full"><h2>Group Exceptions</h2>
<div class="fbar"><label for="et1">Exception<select id="et1"><option value="">All</option></select></label><label for="et2">Environment<select id="et2"><option value="">All</option></select></label><label for="et3">Search<input type="search" id="et3" placeholder="Group, cluster, detail…"></label><label for="et4">Rows<select id="et4"><option value="25">25</option><option value="50">50</option><option value="all">All</option></select></label></div>
<div class="tw sc"><table><thead><tr><th>Group</th><th>Cluster</th><th>Env</th><th>Exception</th><th>Detail</th></tr></thead><tbody id="et"></tbody></table></div>
<div class="foot" id="etf"></div></div>

<div class="grid g2">
<div class="card"><h2>Source Registration Health</h2><p>{{SOURCE_TEXT}}</p><div class="foot">Helios-wide, not filtered by the cluster selection.</div></div>
<div class="card"><h2>Snapshots Past Policy Retention</h2><p>{{SNAPSHOT_TEXT}}</p><div class="foot">Sampled, not filtered by the cluster selection.</div></div>
</div>

<div class="foot">Filtering recalculates the KPIs, coverage by cluster, policy table and exceptions in the page from the data collected at the generated time above; it does not query Helios live. Thresholds are a baseline, not a regulatory requirement. {{SCOPE_NOTE}}</div>
</div>

<script id="data" type="application/json">
{{DATA_JSON}}
</script>
<script>
(function(){
var D=JSON.parse(document.getElementById('data').textContent);
var TYPE={paused:"Paused",sla:"SLA violated",neverrun:"Never run",failed:"Last run failed",repl:"Replication stalled",arch:"Archive failed"};
var sel={};D.clusters.forEach(function(c){sel[c.name]=true});
try{var s=JSON.parse(localStorage.getItem('cpca-sel')||'null');if(s){D.clusters.forEach(function(c){if(c.name in s)sel[c.name]=!!s[c.name]})}}catch(e){}
var $=function(i){return document.getElementById(i)};
function fmt(n){return n.toLocaleString('en-US')}
function el(tag,cls,txt){var e=document.createElement(tag);if(cls)e.className=cls;if(txt!==undefined)e.textContent=txt;return e}
function dur(d){if(d==null)return'-';if(d>=365&&d%365===0)return d/365+' y';if(d>=7&&d%7===0)return d/7+' w';return d+' d'}
function freq(p){if(p.protectOnce)return'Once';if(p.freqH==null)return'-';if(p.freqH>=24&&p.freqH%24===0)return'every '+p.freqH/24+' d';return'every '+p.freqH+' h'}
function sbar(name,a,b,right){var r=el('div','bar');r.appendChild(el('div','n',name)).title=name;
 var t=el('div','t'),tot=a+b;
 var f1=el('div','f');f1.style.width=(tot?a/tot*100:0)+'%';f1.style.background='var(--green-lt)';
 var f2=el('div','f');f2.style.width=(tot?b/tot*100:0)+'%';f2.style.background='var(--red-lt)';
 t.appendChild(f1);t.appendChild(f2);r.appendChild(t);r.appendChild(el('div','c',right));return r}
function buildOpts(){var o=$('opts');o.textContent='';D.clusters.forEach(function(c){
 var l=el('label'),i=document.createElement('input');i.type='checkbox';i.checked=sel[c.name];
 i.addEventListener('change',function(){sel[c.name]=i.checked;save();render()});
 l.appendChild(i);l.appendChild(el('span','',c.name));l.appendChild(el('span','tag',c.connected?'connected':'not connected'));o.appendChild(l)})}
function save(){try{localStorage.setItem('cpca-sel',JSON.stringify(sel))}catch(e){}}
function setAll(fn){D.clusters.forEach(function(c){sel[c.name]=fn(c)});save();buildOpts();render()}
function render(){
 var S=D.clusters.filter(function(c){return sel[c.name]});
 var n=S.length,tot=D.clusters.length;
 $('mssum').textContent=n===tot?'All clusters ('+tot+')':n===0?'No clusters selected':n<=2?S.map(function(c){return c.name}).join(', '):n+' of '+tot+' clusters';
 $('selnote').textContent=n===0?'Select at least one cluster.':'';
 var pr=0,un=0;S.forEach(function(c){pr+=c.protected;un+=c.unprotected});
 $('k1').textContent=(pr+un)?Math.round(pr/(pr+un)*100)+'%':'n/a';$('k1s').textContent=fmt(pr)+' of '+fmt(pr+un)+' objects protected*';
 $('k2').textContent=fmt(un);
 var P=D.policies.filter(function(p){return sel[p.cluster]});
 var PF=P.filter(function(p){var q=($('pq').value||'').toLowerCase(),s=$('ps').value,o=$('po').value,l=$('pl').value;
  return(!s||p.status===s)&&(!o||(o==='yes')===!!p.offsite)&&(!l||(l==='yes')===!!p.lock)&&(!q||(p.name+' '+(p.issues||[]).join(' ')).toLowerCase().indexOf(q)>=0)});
 var cnt={pass:0,warn:0,fail:0};P.forEach(function(p){cnt[p.status]=(cnt[p.status]||0)+1});
 $('k3').textContent=P.length?cnt.pass+' / '+P.length:'n/a';$('k3s').textContent=cnt.fail+' failing, '+cnt.warn+' with warnings';
 var E=D.exceptions.filter(function(e){return sel[e.cluster]});
 var tc={paused:0,sla:0,neverrun:0,failed:0,repl:0,arch:0};E.forEach(function(x){(x.types||[]).forEach(function(t){tc[t]=(tc[t]||0)+1})});
 $('k4').textContent=E.length;$('k4s').textContent=E.length?tc.paused+' paused, '+tc.sla+' SLA violated, '+tc.failed+' last run failed, '+tc.repl+' replication stalled, '+tc.arch+' archive failed, '+tc.neverrun+' never run':'none';
 var ar=0;S.forEach(function(c){ar+=c.atRisk||0});
 $('k2s').textContent='across '+S.filter(function(c){return !c.na}).length+' queried cluster(s); '+fmt(ar)+' protected objects had a failed last run';
 var fd=$('fd');fd.textContent='';
 var fsv=$('fs').value;
 (D.findings||[]).filter(function(f){return !fsv||f.sev===fsv}).forEach(function(f){var r=el('div','find');var c=el('div');c.appendChild(el('span','chip '+f.sev,f.sev));r.appendChild(c);
  var d=el('div');d.appendChild(el('b','',f.title));d.appendChild(el('span','d',f.detail||''));r.appendChild(d);fd.appendChild(r)});
 if(!(D.findings||[]).length)fd.appendChild(el('div','empty','No findings.'));
 var p1=$('p1');p1.textContent='';
 D.envs.slice().sort(function(a,b){return(b.protected+b.unprotected)-(a.protected+a.unprotected)}).forEach(function(e){
  var t=e.protected+e.unprotected;p1.appendChild(sbar(e.env,e.protected,e.unprotected,t?Math.round(e.protected/t*100)+'% of '+fmt(t):'0'))});
 var p2=$('p2');p2.textContent='';
 S.filter(function(c){return !c.na}).sort(function(a,b){return b.unprotected-a.unprotected}).forEach(function(c){
  var t=c.protected+c.unprotected;var bb=sbar(c.name,c.protected,c.unprotected,t?Math.round(c.protected/t*100)+'% of '+fmt(t):'0');if(c.atRisk)bb.title=fmt(c.atRisk)+' protected objects with a failed last run';p2.appendChild(bb)});
 if(!S.some(function(c){return !c.na}))p2.appendChild(el('div','empty','No queried clusters in the selection.'));
 $('p2f').textContent=S.filter(function(c){return c.na||c.partial}).map(function(c){return c.name+': '+(c.na?'n/a ('+c.na+')':'partial ('+c.partial+')')}).join('. ');
 var p3=$('p3');p3.textContent='';
 var mx=Math.max(cnt.pass,cnt.warn,cnt.fail,1);
 [['Pass',cnt.pass,'var(--green-lt)'],['Warn',cnt.warn,'var(--amber-lt)'],['Fail',cnt.fail,'var(--red-lt)']].forEach(function(r){
  var b=el('div','bar');b.appendChild(el('div','n',r[0]));var t=el('div','t'),f=el('div','f');f.style.width=(r[1]/mx*100)+'%';f.style.background=r[2];t.appendChild(f);b.appendChild(t);b.appendChild(el('div','c',String(r[1])));p3.appendChild(b)});
 $('p3f').textContent=D.standards?'Standards: backup at least every '+D.standards.maxFreqHours+' h, retention at least '+D.standards.minRetentionDays+' d.':'';
 var pt=$('pt');pt.textContent='';
 var ord={fail:0,warn:1,pass:2};
 PF.slice().sort(function(a,b){return ord[a.status]-ord[b.status]||b.groups-a.groups}).forEach(function(p){var r=el('tr');
  r.appendChild(el('td','',p.name));r.appendChild(el('td','',p.cluster));r.appendChild(el('td','num',freq(p)));
  r.appendChild(el('td','num',dur(p.retDays)+(p.extDays?' (+ext '+dur(p.extDays)+')':'')));
  r.appendChild(el('td','',p.offsite?'Yes':'No'));r.appendChild(el('td','',p.lock?'Yes':'No'));r.appendChild(el('td','num',String(p.groups)));
  var c=el('td');c.appendChild(el('span','chip '+p.status,p.status));r.appendChild(c);
  r.appendChild(el('td','iss',(p.issues||[]).join('; ')));pt.appendChild(r)});
 if(!PF.length){var e=el('tr'),d=el('td','empty','No policies match the selection and filters.');d.colSpan=9;e.appendChild(d);pt.appendChild(e)}
 $('ptf').textContent=P.length?PF.length+' of '+P.length+' polic'+(P.length>1?'ies':'y')+' shown.':'';
 var et=$('et');et.textContent='';
 var e1=$('et1').value,e2=$('et2').value,e3=($('et3').value||'').toLowerCase(),e4=$('et4').value==='all'?1e9:parseInt($('et4').value,10);
 var EF=E.filter(function(x){return(!e1||(x.types||[]).indexOf(e1)>=0)&&(!e2||x.env===e2)&&(!e3||(x.group+' '+x.cluster+' '+(x.detail||'')).toLowerCase().indexOf(e3)>=0)});
 EF.slice().sort(function(a,b){return(b.types||[]).length-(a.types||[]).length||(a.cluster<b.cluster?-1:1)}).slice(0,e4).forEach(function(x){var r=el('tr');r.appendChild(el('td','',x.group));r.appendChild(el('td','',x.cluster));r.appendChild(el('td','',x.env||''));
  var c=el('td');(x.types||[]).forEach(function(t){c.appendChild(el('span','chip',TYPE[t]||t));c.appendChild(document.createTextNode(' '))});r.appendChild(c);r.appendChild(el('td','iss',x.detail||''));et.appendChild(r)});
 if(!EF.length){var e5=el('tr'),d2=el('td','empty','No exceptions match the selection and filters.');d2.colSpan=5;e5.appendChild(d2);et.appendChild(e5)}
 $('etf').textContent=E.length?'Showing '+Math.min(e4,EF.length)+' of '+EF.length+' matching groups ('+E.length+' with exceptions in the selection), most exception types first.':'';
}
(function(){var tp=$('et1');Object.keys(TYPE).forEach(function(k){var o=el('option','',TYPE[k]);o.value=k;tp.appendChild(o)});
 var ev=[];D.exceptions.forEach(function(x){if(x.env&&ev.indexOf(x.env)<0)ev.push(x.env)});ev.sort().forEach(function(v){var o=el('option','',v);o.value=v;$('et2').appendChild(o)});
 ['fs','ps','po','pl','et1','et2','et4'].forEach(function(i){$(i).addEventListener('change',render)});['pq','et3'].forEach(function(i){$(i).addEventListener('input',render)})})();
$('selall').addEventListener('click',function(){setAll(function(){return true})});
$('selnone').addEventListener('click',function(){setAll(function(){return false})});
$('selconn').addEventListener('click',function(){setAll(function(c){return c.connected})});
document.addEventListener('click',function(e){var d=$('ms');if(d.open&&!d.contains(e.target))d.open=false});
buildOpts();render();
})();
</script>
```
