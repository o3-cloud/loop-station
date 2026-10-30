# Loop Station: Specification

- **Status:** Draft 1
- **Date:** 2026-10-30
- **Owner:** Owen Zanzal
- **Source:** The manager session `4ef2f435-0bbc-42be-a947-f36b383243a0` and the `smartslop-app` loops that it managed from 2026-10-29 23:08 to 2026-10-30 09:41.
- **Requirement syntax:** EARS. Each requirement has one `shall`. The prose outside the requirements uses STE-100.

## 1. Purpose

Loop Station is an AI software factory. It builds one target project with a team of AI agents.

Each agent is a Claude Code session that runs one prompt on a schedule. This specification calls that agent a "worker loop". A manager agent sits above the worker loops. The manager watches the loops, finds problems, and reports to the owner. The owner is the human who sets the direction.

The worker loops do not talk to each other directly. They share files in the target project: a mailbox, the issue list, and the git history.

## 2. Scope

**In scope:**

- The registry of loops, and how to start each loop again.
- The schedule of each loop.
- The mailbox: inbox, issues, ideas, and retrospectives.
- The phase rule, which sets what the loops can build.
- The run of a worker loop, the judge, and the commit rules.
- The manager check, the loop board, and the retrospective.
- The safety rules for shared ports, containers, and git.

**Out of scope:**

- The content of the target project, for example its BDRs and ADRs.
- The hosting of the Claude Code sessions.

## 3. Terms

| Term | Meaning |
| --- | --- |
| Owner | The human who sets the phase, approves changes, and reads the reports. |
| Target project | The git repository that the factory builds. In the source session, this is `smartslop-app`. |
| Worker loop | A Claude Code session with one recurring cron job and one prompt. |
| Manager | The agent that checks all worker loops. The manager is also a loop. |
| Judge | A subagent that a fixer loop starts to verify a fix. The judge does not write the fix. |
| Run | One execution of a loop prompt. |
| Loop registry | The file `loop.json` in the root of the target project. |
| Mailbox | The folder `mailbox/` in the target project, with `inbox/`, `issues/`, `ideas/`, and `retro/`. |
| Current phase | The section "Current Phase" in the `CLAUDE.md` of the target project. |
| Loop board | A published page that shows the state of all loops. |
| Brief | A short status of the factory: a verdict, the good news, the risks, and the next steps. |

## 4. Loop kinds

The source session used these loops. A new target project can use a different set.

| Name | Kind | Interval | Job |
| --- | --- | --- | --- |
| `builder` | Builder | On request | Makes the first version of the project. No loop. |
| `issue-fix` | Fixer | 2 h | Picks one allowed issue, fixes it, and has a judge verify it. |
| `bdr-review` | Reviewer | 2 h | Tests each BDR like a user. Files one issue for each problem. |
| `adr-verify` | Reviewer | 2 h | Checks that each accepted ADR is implemented. Files the gaps. |
| `code-quality` | Reviewer | 6 h | Runs deterministic tools for lint, complexity, and coverage. Files issues. |
| `ideas` | Researcher | 6 h | Writes new feature ideas. Does not build them. |
| `retro` | Retrospective | 2 h | Reviews all loops since the last retrospective. Writes suggestions. |
| `manager` | Manager | 30 min | Checks all loops, updates the loop board, and reports. |
| `progress-brief` | Manager | 4 h | Gives the owner a longer progress brief. |

## 5. Requirements

The system name in every requirement is "the Loop Station". The Loop Station includes the worker loops, the manager, and the judge. A requirement names one of these parts in its condition when only that part acts.

### 5.1 Loop registry (REG)

- **REG-01** The Loop Station shall keep a loop registry in the file `loop.json` in the root of the target project.
- **REG-02** The Loop Station shall record, for each loop, the name, inbox name, purpose, workdir, cron expression, interval, and full prompt.
- **REG-03** The Loop Station shall record, for each loop, the current session ID and the current cron job ID.
- **REG-04** The Loop Station shall record, for each session that has no loop, the session ID, the workdir, and the reason that it has no loop.
- **REG-05** The Loop Station shall record in the loop registry the steps to start a stopped loop again.
- **REG-06** When a loop moves to a new session, the Loop Station shall update the session ID and the job ID of that loop in the loop registry.
- **REG-07** When the loop registry changes, the Loop Station shall commit the loop registry to the target project.
- **REG-08** If the running prompt of a loop differs from the prompt in the loop registry, then the Loop Station shall record the running prompt and a note about the difference.
- **REG-09** The Loop Station shall use absolute paths in each loop prompt that refers to a file outside the workdir of that loop.

### 5.2 Schedule (SCH)

- **SCH-01** The Loop Station shall run each worker loop as a recurring cron job in its own Claude Code session.
- **SCH-02** The Loop Station shall give loops with the same interval different cron minutes.
- **SCH-03** While a loop session is closed, the Loop Station shall show that loop as stopped on the loop board.
- **SCH-04** When a loop misses its expected run time, the Loop Station shall show that loop as "Late" on the loop board.
- **SCH-05** When a cron job is less than 1 day from its expiry, the Loop Station shall tell the owner which loop to start again.
- **SCH-06** If a session closes, then the Loop Station shall keep the loop registry entry, so that the owner can start the loop again from it.

NOTE: In the source session, a cron job is session-only. It stops when its session closes. It expires 7 days after it was created. A resumed session does not start its loops again.

### 5.3 Inbox (INB)

- **INB-01** The Loop Station shall keep owner and manager messages to the loops in `mailbox/inbox/`, one file for each message.
- **INB-02** The Loop Station shall name each message file `NNN-<to>-<short-title>.md`, with the next free number.
- **INB-03** The Loop Station shall give each message the fields To, From, Sent, Status, Message, and Log.
- **INB-04** When a worker loop starts a run, the worker loop shall read each message with the status `Open` that is addressed to its loop name or to `all`.
- **INB-05** When a worker loop acts on a message, the worker loop shall add one line with its name, the time, and the action to the log of that message.
- **INB-06** If the log of a message already has a line from a loop, then that loop shall skip the message.
- **INB-07** If a message asks for work that the current phase does not allow, then the worker loop shall write the reason in the log and skip the work.
- **INB-08** When a loop completes a message that is addressed only to that loop, the Loop Station shall set the status of the message to `Done`.
- **INB-09** While a message to `all` has no log line from one or more addressed loops, the Loop Station shall keep the status of the message `Open`.
- **INB-10** If a worker loop must tell another loop about a problem, then the worker loop shall file an issue instead of an inbox message.
- **INB-11** The Loop Station shall keep every message file after it is done.

### 5.4 Issues (ISS)

- **ISS-01** The Loop Station shall keep all issues in `mailbox/issues/`, one file for each issue.
- **ISS-02** The Loop Station shall name each issue file `NNN-short-title.md`, with the heading `# ISSUE-NNN: Title`.
- **ISS-03** When a loop files an issue, the Loop Station shall give the issue the next free number.
- **ISS-04** When a loop files an issue, the Loop Station shall add a row for the issue to `mailbox/issues/README.md`.
- **ISS-05** Before a loop files an issue, the Loop Station shall search the open and merged issues for the same problem.
- **ISS-06** If an issue with the same problem exists, then the Loop Station shall update the existing issue instead of a new file.
- **ISS-07** When two issues describe the same problem, the Loop Station shall mark one issue `Status: Merged into ISSUE-NNN`.
- **ISS-08** If a review loop files an issue about work that the current phase does not allow, then the Loop Station shall add the line `- **Phase:** later` to that issue.
- **ISS-09** If a file appears in an issue folder other than `mailbox/issues/`, then the manager shall report a collision to the owner.

### 5.5 Ideas and retrospectives (IDR)

- **IDR-01** When the ideas loop runs, the Loop Station shall write the new ideas to a new dated batch file in `mailbox/ideas/`.
- **IDR-02** When the ideas loop runs, the Loop Station shall compare the new ideas with the earlier batches and remove duplicates.
- **IDR-03** When the retro loop runs, the Loop Station shall review all loop runs since the time of the last retrospective.
- **IDR-04** When the retro loop runs, the Loop Station shall write each suggestion to its own file in `mailbox/retro/`.
- **IDR-05** When a retrospective is complete, the Loop Station shall record the end time of that retrospective in `mailbox/retro/README.md`.
- **IDR-06** When the owner asks for a ranking of retro suggestions, the Loop Station shall rank the suggestions by the risk or waste that each one removes and by its effort.

### 5.6 Current phase (PHS)

- **PHS-01** The Loop Station shall keep the current phase in the section "Current Phase" of the `CLAUDE.md` of the target project.
- **PHS-02** The Loop Station shall list in the current phase the work that is allowed now, the work that is allowed after named issues are done, and the work that is not allowed.
- **PHS-03** The Loop Station shall let only the owner change the current phase.
- **PHS-04** When a worker loop picks work, the worker loop shall pick only work that the current phase allows now.
- **PHS-05** While no allowed work is open, the worker loop shall report that and end the run.
- **PHS-06** If the work of the current phase is complete, then the Loop Station shall report that to the owner and wait for the next phase.
- **PHS-07** The Loop Station shall let a loop write ideas and file issues about later work in any phase.
- **PHS-08** If a commit contains work that the current phase does not allow, then the manager shall report that commit to the owner.

### 5.7 Worker run (RUN)

- **RUN-01** When a worker loop starts a run, the worker loop shall read the inbox before its normal work.
- **RUN-02** When a worker loop starts a run, the worker loop shall read the current phase, the issue rules, and the commit rules in `CLAUDE.md`.
- **RUN-03** When the fixer loop starts a run, the fixer loop shall pick exactly one open issue that the current phase allows.
- **RUN-04** While a review loop runs, the review loop shall file issues and leave the product code unchanged.
- **RUN-05** While the code-quality loop runs, the code-quality loop shall use deterministic tools for each finding.
- **RUN-06** When a run ends, the worker loop shall write a run report with the work done and the result of each check.

### 5.8 Judge and verification (VER)

- **VER-01** When the fixer loop completes a fix, the fixer loop shall start a judge subagent to verify the fix.
- **VER-02** The judge shall run each step of the "Verify a Fix" checklist in `CLAUDE.md` itself.
- **VER-03** The Loop Station shall include in the "Verify a Fix" checklist the quality gate (`make check`), a local health check, a compose health check, and a container image health check.
- **VER-04** While one or more checklist steps did not pass, the fixer loop shall keep the issue open.
- **VER-05** If a checklist step is skipped or blocked, then the judge shall record the step as `SKIPPED` and count it as a failure.
- **VER-06** If `docker info` gives no answer in 10 seconds, then the judge shall mark the container steps `SKIPPED`.
- **VER-07** If a fix changes no server code, configuration, Dockerfile, or compose file, then the judge shall run only the quality gate and the local health check.
- **VER-08** When the judge accepts a fix, the fixer loop shall mark the issue `Resolved` with a link to the commit.
- **VER-09** If the judge rejects a fix, then the manager shall report the failed verification on the loop board.

### 5.9 Shared resources (SHR)

- **SHR-01** When a loop starts a server for a test, the loop shall use a free port from 18400 to 18499.
- **SHR-02** If port 8080 is in use, then the loop shall leave the program on port 8080 alone.
- **SHR-03** When a loop starts containers, the loop shall use a compose project name that only that loop uses.
- **SHR-04** When a test ends, the loop shall stop the servers and remove the containers that the test started.

### 5.10 Git (GIT)

- **GIT-01** When a run changes files, the worker loop shall commit the changes to the `main` branch at the end of the run.
- **GIT-02** When a worker loop stages files, the worker loop shall stage only the files that the run changed, by name.
- **GIT-03** If a run changed code, then the worker loop shall run the quality gate before the commit.
- **GIT-04** If the quality gate fails and the run cannot fix it, then the worker loop shall commit and name the failing gate in the commit body.
- **GIT-05** The Loop Station shall write each commit message as a Conventional Commit that names the issue, BDR, or ADR of the change.
- **GIT-06** If `git commit` fails because `.git/index.lock` exists, then the worker loop shall wait and try again without a change to the lock file.
- **GIT-07** If a loop attempts a push, amend, rebase, reset, or branch change, then the Loop Station shall stop that action.
- **GIT-08** If a run changed no files, then the worker loop shall end the run without a commit.
- **GIT-09** If a loop leaves its own changes uncommitted after a run, then the manager shall report that loop to the owner.

### 5.11 Manager check (MGR)

- **MGR-01** The Loop Station shall run a manager check every 30 minutes.
- **MGR-02** When the manager check starts, the manager shall run the collector, which reads the transcript of each loop session.
- **MGR-03** When the manager check runs, the manager shall read the issues, ideas, retro, and inbox folders.
- **MGR-04** When the manager check runs, the manager shall list the running Claude Code sessions.
- **MGR-05** When the manager finds a new loop session, the manager shall add the session to the collector and to the loop registry.
- **MGR-06** When the manager check runs, the manager shall look for collisions: duplicate issue numbers, overwritten files, and an issue folder other than `mailbox/issues/`.
- **MGR-07** When the manager check runs, the manager shall look for late loops, stopped loops, failed verifications, and uncommitted loop work.
- **MGR-08** When the manager check runs, the manager shall report each open inbox message that an addressed loop has not logged.
- **MGR-09** When the manager finds a new problem, the manager shall append a note with the time and the session ID to the manager log.
- **MGR-10** When the manager check ends, the manager shall set the status of the factory to `On track`, `At risk`, or `Off track` from the evidence.
- **MGR-11** When the manager check ends, the manager shall give each good, risk, and next row of the brief a measured number.
- **MGR-12** When the manager check ends, the manager shall send a short status to the owner.
- **MGR-13** If a collision occurs between two loops, then the manager shall send a message to the sessions that collide.
- **MGR-14** If the manager wants to commit, push, or stop a loop, then the manager shall get the approval of the owner first.
- **MGR-15** Every 4 hours, the Loop Station shall give the owner a progress brief.

### 5.12 Collector (COL)

- **COL-01** The collector shall read the transcripts of all loop sessions from each Claude Code project folder of the target project.
- **COL-02** The collector shall write the sessions, runs, issues, tree, and history data as JSON files.
- **COL-03** The collector shall count a run only from a loop prompt of that loop.
- **COL-04** If a session runs more than one loop, then the collector shall count each run under the loop whose prompt started it.
- **COL-05** If a user turn is a subagent result or a skill re-invocation, then the collector shall continue the current run.
- **COL-06** When a session receives a message from another session, the collector shall end the current run.
- **COL-07** The collector shall know each issue status, including `Merged`.
- **COL-08** The collector shall use only the Python standard library.

### 5.13 Loop board (BRD)

- **BRD-01** The Loop Station shall publish the loop board as a private page.
- **BRD-02** The loop board shall show one row for each loop, with its state, last run, next run, and run count.
- **BRD-03** The loop board shall show the report of each run.
- **BRD-04** The loop board shall show the issues with their status, severity, and phase.
- **BRD-05** The loop board shall show the brief with its status and the time of the last check.
- **BRD-06** The loop board shall show the manager log.
- **BRD-07** When the manager check ends, the manager shall update the data of the loop board without a new page version.

### 5.14 Owner control (OWN)

- **OWN-01** The Loop Station shall let the owner send a message to one loop or to all loops through the inbox.
- **OWN-02** When the owner asks to go over the issues, the Loop Station shall present the issues one at a time.
- **OWN-03** When the owner makes a decision, the Loop Station shall record the decision in `CLAUDE.md` or in an ADR.
- **OWN-04** When the owner changes a rule for the loops, the Loop Station shall update `CLAUDE.md` and `AGENTS.md` together.
- **OWN-05** When the owner changes a rule for the loops, the manager shall tell each running loop about the change.
- **OWN-06** If a decision needs the owner, then the Loop Station shall add the decision to a decisions queue and continue other allowed work.

## 6. Evidence from the source session

| Requirement | Event in the source session |
| --- | --- |
| REG-01 to REG-05 | The owner asked for `loop.json`, "in case I need to restart them". |
| SCH-02 | Retro suggestion 01-03: stagger the cron minutes. |
| SCH-06 | Retro suggestion 01-06: loops die with their sessions. |
| INB-01 to INB-11 | Retro suggestion 01-02: one inbox for all loops. |
| ISS-01, ISS-09 | A loop wrote issues to `docs/issues/`. The manager moved them to `mailbox/issues/`. |
| ISS-07 | The manager merged ISSUE-013 and ISSUE-014 into ISSUE-012. |
| PHS-01 to PHS-08 | Retro suggestion 01-09: state what the loops can build. |
| VER-02, VER-03 | ISSUE-026: a fix passed the judge, but `docker compose up` failed. |
| GIT-01, GIT-02 | Retro suggestion 01-01: commit after each fix. |
| COL-03 to COL-06 | The retro loop did not show on the board, and one run showed a reply as its report. |
| MGR-05 | Retro suggestion 01-13: the manager must find new sessions. |
| OWN-06 | Retro suggestion 01-08: a decisions queue. |

## 7. Open questions

1. **Q1** Is Loop Station one factory for each target project, or one factory for many target projects?
2. **Q2** Does Loop Station replace the session-only cron jobs with a durable scheduler, for example launchd? The old `loop-station-v0` used launchd.
3. **Q3** Does the manager start a stopped loop again by itself, or only report it? MGR-14 now says "report only".
4. **Q4** Is the loop board a published page only, or also a local page like `loop-station-v0`?
5. **Q5** Which tool runs the worker loops: only Claude Code, or also the Codex CLI?
6. **Q6** Where is the decisions queue of OWN-06: a folder in the mailbox, or a section of the brief?
