# Coverage Audit Methodology

## Default standards (overridable by the user)
- Maximum backup interval: 24 hours.
- Minimum retention: 30 days.
These are a baseline, not Cohesity or regulatory policy.

## Policy status
- **fail**: the backup interval is longer than the maximum (ProtectOnce policies are exempt from the interval rule), or the longest retention is shorter than the minimum.
- **warn**: no offsite copy, or no DataLock on base retention.
- **pass**: none of the above.
Longest retention = the longest of base retention, extended retention and any replication or archival target retention. A 28-day local policy with a 1-month archive copy passes a 30-day standard. Write short issues such as "Runs every 48 h (limit 24 h)".

## Coverage
- Protection coverage = protected / (protected + unprotected).
- Counts come from the top-level `count` of `dataprotect-search-objects` with `count: 1`. Do not page through objects to count them.
- Environments to check: VMware, Physical, SQL, Oracle, View, O365, AWS, Azure, Acropolis, Kubernetes, HyperV. Drop those with zero objects and keep at most 10.
- Protected but at risk = protected objects whose last protection run failed.
- Object counts per cluster include replicas of the same object on other clusters, so the cluster total is larger than the Helios-wide environment total. Always say so.

## Group exceptions (one row per group)
| Type | Rule |
|---|---|
| paused | Group is paused. Show age in days from the pause time, or from the last edit time if there is no pause time (say so). |
| sla | Last run is marked SLA violated. |
| neverrun | Not paused and has no last run. |
| failed | Last run status is Failed. |
| repl | A replication result of the last run Failed, or the policy has a replication target but the last run has no replication result. |
| arch | An archival result of the last run Failed. Ignore Accepted and Skipped. |

## Source registrations
De-duplicate by `id`. Unhealthy = authentication status is not Finished or a refresh error is present. Stale = last refresh older than 24 hours for auto-refreshing sources (VMware, AWS, Azure, Acropolis, O365, Kubernetes) and older than 7 days for NAS, UDA and physical sources.

## Findings
Write 3 to 8, each with severity (high, medium, low), a title and detail. Typical: large unprotected populations (with counts and environment), policies failing the standards, groups paused for a long time, SLA violations, unhealthy registrations, clusters that could not be queried.

## Clusters not assessed
Clusters not connected to Helios cannot be queried: list them as "not connected to Helios". If a connected cluster answered the object counts but its groups, policies or registrations timed out, keep its counts and say the rest is partial.
