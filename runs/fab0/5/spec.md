# check-agents.py drives a 0.10 run

## Intent

Whoever changes an agent, a skill or the wiring of the `story`, `feature` or `delivery` example needs the live agent check to tell them whether the examples still work end to end. Since 0.10 the check fails at its first step: it drives a 0.9 run that reads a run store, merges review pull requests by hand and stops before the merge into `main`. So changes to the examples' agents go unchecked. It must drive a 0.10 run again, act only where the examples name a person, and judge the outcome from the tracker and the repositories.

## Scope

- In: `scripts/check-agents.py` in fab0-examples, for `--example story` and `--example feature`, against the fake tracker (`FAB0_GH_FAKE`) and local bare origins, as before.
- In: what it does in place of a person: approving a spec, a requirements or a wrap-up pull request, confirming a task list or a plan at the prompt, and answering an agent's questions.
- In: the script's docstring, and the README and `CLAUDE.md` of fab0-examples where they describe its usage or what it checks.
- Out: changes to the examples' agents, skills, wiring or versions, and to the engine in fab0-core. A defect the check finds there becomes an issue of its own in the repository that owns it.
- Out: `scripts/check-examples`. The check may reuse its functions as they are.
- Out: a person's other decisions: asking for changes, rejecting a plan or a hand-back, deciding an escalation. The check never makes them.
- Out: GitHub behaviour the fake lacks, apart from the merge of a run pull request (S8). For example, a run pull request's merge closing the `story` example's Story isn't checked.
- Out: the `triage` example, real GitHub, running the check in CI, and the docs in `fab0-dev/docs`, which don't describe the script.

## Scenarios

### S1: the story workspace is set up as its README says

- Given `--example story`
- When the check sets up its workspace
- Then the home checkout `hotel` holds the `story` example copied as its README says, with `OWNER` replaced by `acme` and `REPO` by `hotel` everywhere in `.fab0/`, and `po-user` as the product owner
- And the runs repository `hotel-runs` is a sibling checkout, and each repository has a local bare `origin`
- And the fake tracker holds the Story `acme/hotel#1` with the booking intent, and `fab0-resolve validate` passes in the home checkout before any session starts

### S2: an engine or wiring that doesn't validate stops the check before it spends anything

- Given no engine at the sibling fab0-core checkout or at `$FAB0_CORE`, or a `validate` that fails in any checkout of the workspace
- When the check starts
- Then it prints the reason (the missing engine, or `validate`'s errors), starts no Claude Code session, exits 1, and keeps the workspace when there is one

### S3: the first session opens the run, and the spec waits on its owner

- Given the story workspace
- When the check runs `/fab0:run #1 --profile cheap` in the home checkout
- Then the check takes the run id from the run's Pipeline Run in `acme/hotel-runs`, which is a sub-issue of the Story
- And the `spec` unit waits on its human gate, with an open pull request in `acme/hotel-runs` that adds `runs/hotel/<run>/spec.md` and no task list
- And `acme/hotel` has no pull request and no new commit yet

### S4: the product owner approves the spec as they would on GitHub

- Given the spec waits on its human gate
- When the check acts as the product owner
- Then it comments `/approve` as `po-user` on the spec's pull request, timed after that pull request's head commit, and merges, pushes and approves nothing else
- And in the next session fab0 merges the spec's pull request, the planner writes `runs/hotel/<run>/tasks.yaml`, and the `plan` unit waits for a confirmation at the prompt

### S5: an agent's questions get one plain answer

- Given a unit owned by a named person waits on questions after a session
- When the check acts as that person
- Then it answers once, where the run takes answers (the unit's pull request, or the run's item when it has none), with a fixed reply that tells the agent to decide and say what it chose
- And the next session reworks the unit with that answer

### S6: the product owner confirms the task list, and only the product owner does

- Given the `plan` unit waits for a confirmation at the prompt
- When the check confirms it as `po-user`, in the home checkout, between sessions
- Then the confirmation is accepted, and the next session creates the tasks
- But when the confirmation is refused because the plan was decided already, the check fails with "a session confirmed for the person"

### S7: every task is implemented, reviewed and merged into the run branch

- Given a confirmed task list
- When the following sessions run
- Then each task of `tasks.yaml` has one Task sub-issue of the Story, and every `implement` unit is done: reviewed by the `reviewer` agent and merged into the run branch by `apply`

### S8: the check stands in for GitHub's merge of a run pull request, and for nothing else

- Given `apply` has marked a run pull request merged in the fake tracker, in `acme/hotel` or `acme/hotel-runs`
- When the check acts between sessions
- Then it merges that pull request's branch into the base on that repository's `origin`, as GitHub would
- And it touches no run pull request the fake doesn't show merged, and merges no other branch

### S9: the story run ends done, and its issues say so

- Given every task is merged and the run pull requests are merged (S8)
- When the next session runs
- Then `fab0-resolve status <run>` says the run is done
- And every Pipeline Run and Pipeline Step of the run carries the run id as its `Run ID`, and no other value; all of them are closed; every Task sub-issue is closed
- And `apply` marked each run pull request merged: the check never changes a pull request's state in the fake tracker

### S10: the product repository holds only the tasks' changes

- Given the story run is done
- When the check inspects the repositories
- Then the tests pass on `main` of `acme/hotel`'s `origin`
- And every file that differs between the starting commit and that `main` is claimed by a task of `tasks.yaml`
- And no commit on any branch of `acme/hotel` is empty, and no spec, task list or file of the run's state is in it
- And `main` of `acme/hotel-runs`'s `origin` holds `runs/hotel/<run>/spec.md` and `runs/hotel/<run>/tasks.yaml`

### S11: a run that stops on what no named person decides fails the check

- Given either example
- When, after a session, a unit is escalated, blocked by a failure, or waits on a person for anything S4, S5, S6 or S15 doesn't cover (questions a second time on the same unit included)
- Then the check starts no further session, fails naming each such unit and its reason, exits 1 and keeps the workspace

### S12: a run that doesn't finish within the bound fails the check

- Given either example
- When the bound on sessions is reached and the run isn't done
- Then the check fails with the run's position as `fab0-resolve status` gives it, exits 1 and keeps the workspace

### S13: the composition workspace is set up as each README says

- Given `--example feature`
- When the check sets up its workspace
- Then `product` holds the `feature` example, and `service` and `web` beside it each hold a copy of `delivery`, with `OWNER`, `REPO` and `PRODUCT-runs` replaced as each README says, and `po-user` in every role
- And the runs repository `product-runs` is a sibling checkout, and each of the four repositories has a local bare `origin`
- And the fake tracker holds the Feature `acme/product#1` with the Story `acme/product#2` as its sub-issue
- And `validate` passes in each of the three factories' checkouts, and the parent's checks both connections

### S14: each factory runs in its own session and checkout, as its people would

- Given the composition workspace
- When the check starts the parent's run with `/fab0:run #2 --profile cheap` in `product`
- Then the run id is the parent's Pipeline Run number, and each child run is taken up only by a session in its own factory's checkout
- And after that, each factory gets a session in turn whenever it has work to do, one session at a time, until the parent's run is done or the bound is reached (S12)

### S15: the composition's people act where the examples name them

- Given a composition session has ended
- When a unit waits on a named person
- Then the check approves the requirements' and the wrap-up's pull requests as the product owner, as in S4; confirms each team's plan as that team's developer, in that team's checkout, as in S6; answers questions as in S5; and stands in for run pull request merges as in S8
- And it decides nothing else

### S16: the composition's runs form one tree with one id, closed at the end

- Given the parent's run is done
- When the check reads the tracker
- Then the parent's Pipeline Run is a sub-issue of the Story and holds the service's and the web's Pipeline Runs as sub-issues
- And every Pipeline Run and Pipeline Step of the three factories carries the parent's run id as its `Run ID`, and all of them are closed
- And `acme/service` and `acme/web` hold no issue

### S17: the parent accepted what both teams handed back

- Given the parent's run is done
- When the check reads the parent's Pipeline Steps
- Then the parent's acceptor approved the service's hand-back, its contract included, and the web's hand-back

### S18: each team is built against what the parent accepted

- Given the parent's run is done
- When the check reads the child runs' Pipeline Steps
- Then each team's plan read the requirements at the version the product owner approved
- And every web implementation attempt that was merged read the contract at the version the parent accepted
- But a merged web attempt that read another version of the contract, or none, fails the check

### S19: the wrap-up closes the Story, and the Feature is closed or has follow-ups

- Given the parent's run is done
- When the check reads the tracker
- Then the Story `acme/product#2` is closed
- And the Feature is closed, or holds follow-up Stories under it besides `acme/product#2`

### S20: the composition's repositories hold only the tasks' changes

- Given the parent's run is done
- When the check inspects the repositories
- Then `acme/product` has no new commit, no run branch and no pull request
- And every file that differs between the starting commit and `main` of `acme/service`'s and `acme/web`'s `origin` is claimed by a task of that team's task list
- And no commit on any branch of the four repositories is empty, and no file of a run's state is in any of them

### S21: the usage and the result read as before

- Given any of `--example story|feature`, `--model`, `--budget` and `--keep`, with the same defaults as today
- When the check ends
- Then it prints each expectation as `ok` or `FAIL`, each session's cost and the total, and exits 0 when all passed or 1 otherwise
- And it removes the workspace only when all passed and `--keep` isn't given, and otherwise prints where it kept it

### S22: the check never reaches GitHub

- Given a machine where `gh` is logged in to GitHub
- When the check runs either example
- Then every session and every engine command it starts uses the fake tracker and the local origins, and no issue, comment, pull request or push reaches GitHub

### S23: the docs say what the check does now

- Given the rewritten check
- When a maintainer reads the script's docstring, the README's Develop section and `CLAUDE.md`'s Checks section
- Then all three give the same command line for both examples, and the docstring describes the 0.10 run: the approval, the confirmation, the run's merge into `main`, and the outcome read from the tracker and the repositories

## Constraints

- Runs as `CLAUDE.md` says, with `python3`, a logged-in `claude`, and `uv` for the engine. It adds no dependency.
- Uses the engine of a sibling fab0-core checkout or `$FAB0_CORE`, at 0.10 or later. An installed fab0 plugin's `bin` never comes before it on `PATH`.
- Costs stay bounded: every session runs on `--profile cheap`, under `--max-budget-usd` and a timeout, and the number of sessions is bounded.
- Writes nothing outside its temporary directory.
- Reads the outcome from the fake tracker's file, git and `fab0-resolve`'s read-only commands, never from a session's own report.

## Decisions

- Approving a pull request: a `/approve` comment by `po-user`, timed after its head commit. fab0 opened the pull request with that login, and GitHub doesn't let anyone approve a pull request they opened; the examples' READMEs say to comment `/approve` instead.
- Confirming at the prompt: `fab0-resolve confirm <run> <unit>` in that factory's home checkout, between sessions. A headless session has nobody to ask, and a person's yes at the prompt runs that same command.
- One tracker login, `po-user`, in every role: the fake tracker has a single user.
- The run pull request's merge on `origin` is the check's (S8): the fake marks it merged but doesn't move the base, and the engine counts a run done only once its run branch is in the base.
- Sessions run one at a time, each in its factory's checkout. The bound is a fixed number of rounds, eight as today.
- The answer to questions is one fixed reply, the same for every unit: deciding product questions is the person's, and the check doesn't play at it.

## Open questions

None
