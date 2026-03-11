# Asynchronous Issue Triage

This proposal introduces an asynchronous triage process for GitHub issues, allowing maintainers and component owners to triage issues outside of the community call using emoji reactions.
The process starts as a manual workflow with the goal of automating it in the future.

## Current situation

Currently, issue triage happens synchronously during community calls.
This takes up valuable call time and can delay triage for issues that are straightforward to evaluate.
Issues may sit untriaged until the next community call, even when maintainers already have a clear opinion on them.

## Motivation

Asynchronous triage allows maintainers to review and vote on issues at their own pace, freeing up community call time for issues that genuinely need discussion.
It also reduces the triage backlog by enabling continuous progress between calls.

## Proposal

> [!IMPORTANT] 
> An issue is eligible for asynchronous triage when it is labeled `needs-triage`.
> All new issues should be marked with the label `needs-triage` when/after they are created.

### Voting mechanism

Maintainers and component owners with merge rights in the given repository indicate their opinion using emoji reactions on the issue:

| Reaction | Meaning                                                                         |
|----------|---------------------------------------------------------------------------------|
| 👍       | We should keep this issue                                                       |
| 👎       | We should close this issue (with an explanatory comment)                        |
| 👀       | I saw this but want to discuss it further (ideally with an explanatory comment) |

Any user is encouraged to share their opinion.
But only votes from maintainers and component owners with merge rights in the given repository count toward the triage decision.

### Decision rules

After at least **5 days** since the last major comment on the issue, the following rules apply:

- **Approved (keep):** 
No 👀 and no  👎, and at least 3 👍 reactions. 
The issue is considered triaged and should be kept. 
The `needs-triage` label is removed.
- **Rejected (close):** 
No 👀 and no 👍, and at least 3  👎 reactions. 
The issue is considered triaged and rejected. 
It should be closed with an explanatory comment or with acknowledgment of another maintainer explanation.
- **Needs discussion:** 
Any other mix of reactions (e.g., presence of 👀, mixed votes, or fewer than 3 votes). 
The issue remains labeled `needs-triage` and is discussed at the next community call.

### Additional labels

- **`needs-proposal`:** 
Can be added at any point by anyone who thinks the issue requires a formal proposal. 
If the issue is approved via async triage, the label will remain.
- **`good-start` / `help-wanted`:** 
The person marking the issue as triaged (or anyone else later) can add these labels if desired, but they should be ready to provide guidance to whoever wants to work on it.

### Pre-call check

Until automation is in place, the person running the community call should do a manual check of `needs-triage` issues before the call.
This ensures that issues which reached consensus asynchronously are cleaned up, and remaining issues are queued for discussion.

### Future automation

A bot, scanner, or cron job (to be investigated separately) could automate the process:

- Monitor `needs-triage` issues for vote counts after the 5-day (i.e., 120 hours) window
- Automatically remove the `needs-triage` label when consensus is reached (based on decision rules stated above)

## Affected/not affected projects

This proposal affects the Strimzi project's issue triage workflow across all repositories.
No code changes are required initially (i.e., this is a process change).
Future automation tooling may be developed separately.

## Compatibility

This process is additive and does not replace synchronous triage during community calls.
Issues that do not reach async consensus are still discussed during the call as before.

## Rejected alternatives

### Using comments or commands instead of emoji reactions

Using comments or bot commands (e.g., `/strimzi-triage +1`) was considered but rejected in favor of emoji reactions only.
Emoji reactions provide a quicker, more visible way to see the current vote count at a glance without cluttering the issue with additional comments.

### 72-hour triage window

An initial 72-hour window was discussed but extended to 5 days to give maintainers enough time to review issues, especially across different time zones and work schedules.