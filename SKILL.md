---
name: council
description: Run a multi-lab review council on work where quality is a judgment call, such as a pull request, a plan, a README, product copy, UI or ads. Judges from different labs score it blind, every finding goes into a ledger, and the main agent fixes until the ledger is clean. Use only when the user says council or asks for a multi-model or multi-provider review. Not for routine reviews, throwaway work or changes that tests already settle.
---

# Council

Judges from different labs review the same work independently. Their errors are less correlated than one model's, so together they catch more. Scores show direction. The ledger decides when the work is done. Agreement between judges is a signal, never proof.

The main agent runs the council and does all the editing. Judges only read and report.

## Modes

- **solo:** one judge, from a lab other than the main agent's. Small diffs and quick checks.
- **full:** every seat. Real pull requests, plans and decisions.
- **references:** full, plus a blind comparison against reference examples of the same kind of work. Use for subjective work such as READMEs, landing pages, UI copy, ads and visual design.

Infer the mode from the request. Say which mode you chose and why in the intake.

## Seats

Call `orchestrator_capabilities` at the start of every run. Use `search` when a model is missing from the short list. Defaults as of 2026-11:

| seat | provider | model | effort | gates the stop |
|---|---|---|---|---|
| 1 | `claudeAgent` | `claude-opus-5-5` | `xhigh` | yes |
| 2 | `codex` | `gpt-6.1-sol` | `xhigh` | yes |
| 3 | `antigravity` | `gemini-3.8-flash-high` | default | no |

- Seat 3 is a weaker model kept for lab diversity. Its findings count when they carry evidence. Its scores are reported but do not gate the stop.
- Do not seat Grok, Muse Spark or free OpenCode models unless the user asks. Models billed through OpenRouter, such as Muse Wonder, cost extra money and need the user's approval.
- If a seat cannot run, say so and run with the seats that can. Never substitute a model silently.
- In solo mode, use seat 2 when the main agent is Claude and seat 1 when it is not.
- When the main agent and a judge come from the same lab, expect self-preference bias. The blind labels in references mode reduce it.

## Intake

Before spending anything, send one message with these questions, each with a recommended answer. Skip any that the request already answers.

1. **Scope:** the exact files or surfaces, and what is out of scope.
2. **References** (references mode): 3 to 5 examples the user admires, or a proposed list for the user to choose from. The user's taste picks the final set.
3. **Categories:** 4 to 6 categories, each with a one-line definition and anchors for 3, 6 and 9. The categories are frozen once approved, so scores stay comparable across rounds. Example for a README: clear in ten seconds, shows why it is worth using, install friction, scannability, voice.
4. **Owner calls:** decisions the judges may not overturn, such as "keep the moon screenshot" or "English only".
5. **Budget:** default 5 rounds. Give a rough cost: each full round is about three strong reviewer runs.

Wait for answers. A short "go" accepts every recommendation.

## Setup

1. Fix known issues before the first round. Judges should not spend a round on problems you already know about.
2. Create a workspace in the scratchpad: `council/<subject>/` containing `candidate/`, `references/`, `rounds/`, `ledger.md` and `mapping.json`.
3. Write the hard checks as commands where possible, and run them on the baseline:
   - **code:** the repository's build, tests and linters
   - **prose:** every factual claim traced to a file, measurement or source; links and images resolve
   - **UI:** contrast ratios, string length against the space available, screenshots render
4. In references mode, capture the candidate and every reference in the same format at the same size, for example the text plus a screenshot rendered at 900 px. For Markdown, render the candidate and the references with the same renderer, such as `gh api markdown` or the GitHub page itself.

## A round

1. In references mode, copy the entries into `rounds/<n>/` as A, B, C and so on, shuffled freshly each round. Record the mapping in `mapping.json` and never show it to a judge. Remove file names and paths that reveal which entry is the candidate. A project name inside the text cannot be hidden and that is acceptable.
2. Start one fresh `delegate_task` per seat, in parallel, each with its own stable `clientRequestId`. Judges are read only and must not delegate further. Give each judge only the round folder, or for code the exact candidate revision, never the working repository in references mode.
3. Each brief contains:
   - the user's goal, quoted, with your additions labeled as yours
   - the mode, the categories with their anchors, and the owner calls
   - from round 2 on, the open ledger rows and the rejected rows with their reasons, so judges do not raise them again without new evidence
   - never the target, previous scores or other judges' output
4. Ask each judge to return:
   - in references mode, a score from 1 to 10 for every entry in every category, each with a one-line reason, plus an overall score. The judge is not told which entry is the candidate, so it scores all of them.
   - findings, each with a location (file and line, or a measurable value), the claim, the impact, and a severity of blocker, major, minor or taste
   - in references mode, what the best entry does that the others do not
   - "no score" wherever it cannot judge. Never invent a score for a judge that did not give one.

## Ledger

`ledger.md` holds one row per finding. Deduplicate across seats and record which seats raised each row. A finding raised by several labs is a priority signal, not proof.

| id | round | seats | claim | evidence | status | verification |
|---|---|---|---|---|---|---|

Status is one of:
- **open:** plausible and unresolved
- **fixed and verified:** the change addresses it and a check passed on the current candidate
- **rejected with evidence:** contradicted by a reproduction, a source, a measurement or an owner call. Record the reason.
- **deferred:** valid but out of scope, or waiting on the user. A deferred blocker prevents the stop.
- **declined as taste:** a preference that does not affect the categories or the goal. Say why, without claiming it was disproved.

## Fixing

- Edit only rows that are open and valid. Use whatever installed skill fits the medium: the writing skills for prose, the design skill for UI and in-app text, normal engineering practice for code. Choose the skill by its role, not by a fixed name.
- Owner calls override every finding.
- Run the hard checks after each round's fixes. A failed check is a blocker regardless of scores.

## Stop

Stop when all of these hold:
- every blocker and major row is fixed and verified or rejected with evidence
- every hard check passes on the current candidate
- in references mode, each gating seat scores the candidate overall at least 0.5 above the best reference, in two consecutive rounds. Judge the gap, not the absolute score, because scores on unchanged references drift between rounds.

Also stop when:
- a round produces only minor and taste findings. Report done.
- the round budget is reached. Report where things stand and hand the final call to the user.
- the same dispute repeats without new evidence. Report it as unresolved.

Never run an unchanged round just to get agreement, and never tell a judge the target.

## Delivery

For work in a repository: one branch with one pull request. Ask before pushing or opening it unless the user already authorized that. The pull request description contains:
- before and after, as text or screenshots
- a `## Council review` table with one row per round, one column per seat with its overall score (and the best reference's score in references mode), and the fixes that round produced
- a ledger summary: counts by status, plus every rejected or deferred row with its reason
- the owner calls and the hard check results
- the cost, when the host reports it

For work outside a repository, give the same report in chat.

Finish with the verdict: **ready for human review**, **changes required** or **verification incomplete**.

Adapted from the `council` skill in [t3dotgg/tstack](https://github.com/t3dotgg/tstack) (MIT), with the ledger and stop rules from a separate review skill.
