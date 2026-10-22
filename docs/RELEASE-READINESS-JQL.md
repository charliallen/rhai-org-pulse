# Release Readiness JQL Reference

Temporary reference for `jira/release-readiness/fetch_release_metrics.py`.

## Shared release version clause

For `rhoai-3.5.EA2`, the validated Jira version clause is:

```jql
(fixVersion IN ('rhoai-3.5.EA2', '3.5 EA2 RHOAI RELEASE')
 OR affectedVersion IN ('rhoai-3.5.EA2', '3.5 EA2 RHOAI RELEASE')
 OR 'Target Version' IN ('rhoai-3.5.EA2', '3.5 EA2 RHOAI RELEASE'))
```

The script uses values from `JIRA_VERSIONS` when configured; otherwise it derives and validates version variants against the `RHOAIENG` project.

## Release Cycle Metrics

| Value | Source |
| --- | --- |
| Code Freeze | Product Pages API; the checked-in release-variable YAML fills the field when Product Pages has no value. No JQL. |
| RC1/RC2 Build Received | Manual override → Stage Promoter → Product Pages schedule (checked-in YAML fills missing fields) → if those have no usable date, Jira: `RCn Push to Stage` resolved date, then `Prepare RCn` if needed → `null`. |
| Test Started | Same release-phase Epic query; first Jira changelog transition to `In Progress` on the test Epic. |
| Test Finished | Same query; test Epic `resolutiondate`, falling back to `updated` only when that test Epic is Done. |
| TFAs Passed | TFA query below; latest `updated` when every task is In Progress or Done. |
| TFAs Triaged | TFA query below; latest `updated` when every task is Done. |
| Blockers Resolved | Resolved-blocker query below; latest `resolutiondate`, falling back to `updated`. |
| Working-day values | Calculated locally from the dates; no JQL. |

When Code Freeze and Test Started are known, Build Received candidates must be
on or after Code Freeze and on or before that RC's Test Started date. A
candidate outside those inclusive bounds is skipped and the extractor tries
the next candidate or source.

`ACTUAL_BUILD_DATES` is an optional JSON environment variable keyed by release
and phase, for example `{"rhoai-3.5.EA2":{"RC1 Builds Testing":"2026-07-08"}}`.
If it is unset or empty, the manual-override source is skipped. When set, it
takes precedence over the other Build Received sources for that phase.

Stage Promoter dates come from successful matching RC runs of the
`stage-promoter.yaml` GitHub Actions workflow in
`red-hat-data-services/rhods-devops-infra`. The extractor reads the release and
build type from the successful `show-info` job log and uses the workflow
completion date. For each RC it selects the latest matching date on or after the
known lower bound—Code Freeze for RC1, or the latest earlier RC's Test Started
date for later RCs—and on or before that RC's Test Started date. If no lower
bound is known, only the Test Started upper bound applies. This source is
skipped if `GITHUB_TOKEN`/`GH_TOKEN` is unset or the API lookup fails.

The release-variable YAML is checked in under the extractor repository's
`jira/` directory; for example, release `rhoai-3.6.EA2` reads
`jira/jira-release-variables-3.6.0-ea2.yaml`. The extractor merges its RC
milestone fields into the schedule only when Product Pages has no value for
those fields. In that 3.6 EA2 file, `rc1_build_received_date: "2026-09-29"`
maps to `release_schedule.rc1_build_date`; its adjacent comment cites
`#rhoai-build-notifications`. This is version-controlled input read by the CI
extractor, not a file generated on the dashboard host.

Release initiative Epics (test dates and RC Build Received dates):

```jql
(parent = <INITIATIVE_KEY>
 OR "customfield_10014" = <INITIATIVE_KEY>)
AND issuetype = Epic
```

This query returns all Epics under the release initiative. The extractor gets
Test Started and Test Finished from the `RCn Builds Testing` Epics; those test
dates never supply Build Received.

For Build Received, Jira is checked only when the override, Stage Promoter,
and Product Pages/YAML have no usable date. For RC1, the extractor checks:

1. The Jira Epic whose summary contains `RC1 Push to Stage`. Use its
   `resolutiondate` (the date Jira recorded for resolving the issue) if it is
   on or after Code Freeze and on or before Test Started, when those dates are
   known.
2. If that date is missing or outside those limits, check the Epic whose
   summary contains `Prepare RC1`, using the same date rule.

Repeat the same order for RC2. If neither Jira date qualifies, Jira supplies no
Build Received date. The extractor does not substitute a test Epic's status or
completion date, or a build Epic's `updated` date.

TFA milestones:

```jql
project = RHOAIENG
AND <VERSION_CLAUSE>
AND summary ~ "TFA Sign-Off"
AND issuetype NOT IN (Epic, Initiative)
```

Open blockers, used before calculating the resolved date:

```jql
project in (RHAI, RHAIENG, RHOAIENG)
AND (labels not in (RHOAI-releases, RHOAI-internal, devtestops-service, test-failed, test-skipped)
     OR labels IS EMPTY)
AND (component not in (Documentation, PXE) OR component is EMPTY)
AND status not in (Closed, Resolved)
AND ('Release Blocker' != Rejected OR 'Release Blocker' is EMPTY)
AND <BLOCKER_VERSION_CLAUSE>
AND priority in (Blocker)
AND (labels IS EMPTY OR labels not in (prerelease-cve-remediation))
```

Where `<BLOCKER_VERSION_CLAUSE>` is:

```jql
(fixVersion IN ('rhoai-3.5.EA2', '3.5 EA2 RHOAI RELEASE')
 OR 'Target Version' IN ('rhoai-3.5.EA2', '3.5 EA2 RHOAI RELEASE')
 OR (affectedVersion IN ('rhoai-3.5.EA2', '3.5 EA2 RHOAI RELEASE')
     AND fixVersion IS EMPTY AND 'Target Version' IS EMPTY))
```

Resolved blockers:

```jql
project in (RHAI, RHAIENG, RHOAIENG)
AND (labels not in (RHOAI-releases, RHOAI-internal, devtestops-service, test-failed, test-skipped)
     OR labels IS EMPTY)
AND (component not in (Documentation, PXE) OR component is EMPTY)
AND <BLOCKER_VERSION_CLAUSE>
AND priority in (Blocker)
AND (labels IS EMPTY OR labels not in (prerelease-cve-remediation))
AND statusCategory = Done
```

## Other dashboard data

Release-activities Initiatives:

```jql
issuetype = Initiative
AND <VERSION_CLAUSE>
AND summary ~ "release activities"
AND status NOT IN (Cancelled)
```

Initiative or Epic children:

```jql
parent = <PARENT_KEY>
OR "customfield_10014" = <PARENT_KEY>
```

Linked TFA or blocker issues:

```jql
key IN (<ISSUE_KEY_1>, <ISSUE_KEY_2>, ...)
```

Component TFA counts:

```jql
project = RHOAIENG
AND component = "<COMPONENT>"
AND <VERSION_CLAUSE>
AND summary ~ "TFA Sign-Off"
AND issuetype NOT IN (Epic, Initiative)
```

Failed tests:

```jql
project in (RHAI, RHOAIENG)
AND component = "<COMPONENT>"
AND <VERSION_CLAUSE>
AND labels = "test-failed"
```

Skipped tests:

```jql
project in (RHAI, RHOAIENG)
AND component = "<COMPONENT>"
AND <VERSION_CLAUSE>
AND labels = "test-skipped"
```

Open issues to validate:

```jql
project in (RHAI, RHAIENG, RHOAIENG)
AND (labels not in (RHOAI-releases, RHOAI-internal, devtestops-service, test-failed, test-skipped)
     OR labels IS EMPTY)
AND (component not in (Documentation, PXE) OR component is EMPTY)
AND status not in (Closed, Resolved)
AND ('Release Blocker' != Rejected OR 'Release Blocker' is EMPTY)
AND <OPEN_ISSUES_VERSION_CLAUSE>
```

Where `<OPEN_ISSUES_VERSION_CLAUSE>` is:

```jql
(fixVersion IN ('rhoai-3.5.EA2', '3.5 EA2 RHOAI RELEASE')
 OR 'Target Version' IN ('rhoai-3.5.EA2', '3.5 EA2 RHOAI RELEASE'))
AND ('Target Version' IS EMPTY
     OR 'Target Version' IN ('rhoai-3.5.EA2', '3.5 EA2 RHOAI RELEASE'))
```

## Generated Jira filter links

These JQLs are embedded in dashboard links. They are generated for navigation and are not all executed by the extractor.

Component TFA link:

```jql
project = RHOAIENG
AND component = "<COMPONENT>"
AND <VERSION_CLAUSE>
AND summary ~ "TFA Sign-Off"
```

Component failed/skipped links use the same failed/skipped queries above, scoped by component:

```jql
component = "<COMPONENT>"
```

Component execution link (aggregated across all test phases):

```jql
parent in (<TEST_EXECUTION_EPIC_KEY_1>, <TEST_EXECUTION_EPIC_KEY_2>, ...)
AND component = "<COMPONENT>"
```

Overall work filters:

```jql
(parent IN (<INITIATIVE_KEY_1>, <INITIATIVE_KEY_2>, ...)
 OR "Epic Link" IN (<INITIATIVE_KEY_1>, <INITIATIVE_KEY_2>, ...))
```

The work-in-progress and work-done links append, respectively:

```jql
AND statusCategory = "In Progress"
```

```jql
AND statusCategory = "Done"
```

Linked TFA-in-review and blocker links:

```jql
key IN (<TFA_KEY_1>, <TFA_KEY_2>, ...)
AND status = "In Review"
```

```jql
key IN (<BLOCKER_KEY_1>, <BLOCKER_KEY_2>, ...)
AND statusCategory != Done
```

Gate links for an individual Epic:

```jql
parent = <EPIC_KEY>
OR "Epic Link" = <EPIC_KEY>
```

The Test Execution gate instead uses:

```jql
parent in (<TEST_PHASE_EPIC_KEY_1>, <TEST_PHASE_EPIC_KEY_2>, ...)
AND issuetype NOT IN (Epic, Initiative)
```

`customfield_10014` is the Epic Link field in the Red Hat Jira instance. Team IDs used by component queries are instance-specific.

---

Assisted-by: Codex
