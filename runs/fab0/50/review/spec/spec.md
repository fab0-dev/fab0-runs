# apply goes on when GitHub's branch is ahead of the local one

## Intent

A person decides a human gate by merging a unit's pull request on GitHub, which moves a branch on GitHub past the copy in the local checkout. `apply` then tries to push the stale local branch, GitHub refuses it as not a fast-forward, and the run stops until someone moves the branch by hand. A branch GitHub already has in full needs no push: the run should go on by itself.

## Scope

- In: every branch `apply` pushes (run branches and unit branches), when GitHub's copy contains every commit of the local one. The local branch is brought forward to GitHub's, within D42, and nothing is pushed.
- In: the explanation of branches in the docs says so.
- Out: diverged branches, where each side has commits the other lacks. `apply` still stops with the push error.
- Out: fetching. The session fetches before `apply` (run skill, step 2); a branch that moves on GitHub after that fetch is not detected here.
- Out: the run store's own push, which already handles a remote ahead of it.

## Scenarios

### S1: the run's pull request opens after a unit merged on GitHub

- Given a run branch whose copy on GitHub has a commit the local one lacks (a person merged a unit's pull request into it on GitHub), the local one has no commit GitHub lacks, the branch is not checked out, and the session has fetched
- When `apply` opens the run's pull request in that repository
- Then the pull request is opened, `apply` reports no failure, and the local run branch points at the same commit as GitHub's

### S2: a unit branch behind GitHub

- Given a unit branch whose copy on GitHub is ahead of the local one and contains all of it, not checked out
- When `apply` pushes it, to open the unit's pull request or before merging it into its parent
- Then `apply` goes on as if the push had succeeded, and the local unit branch points at GitHub's commit

### S3: the local branch has new commits

- Given a branch whose local copy has commits GitHub lacks, and GitHub has none the local one lacks
- When `apply` pushes it
- Then GitHub's branch moves to the local commit, as today

### S4: both are at the same commit

- Given a branch whose local copy and GitHub's are the same commit
- When `apply` pushes it
- Then no branch moves and `apply` goes on

### S5: diverged branches still stop the run

- Given a branch where the local copy and GitHub's each have a commit the other lacks
- When `apply` pushes it
- Then `apply` fails with an error that names the repository and the branch, and neither the local branch nor GitHub's moves

### S6: the branch is checked out in a clean checkout

- Given a branch behind GitHub's, as in S1 or S2, checked out in a worktree with no changes
- When `apply` pushes it
- Then that checkout is fast-forwarded to GitHub's commit, and `apply` goes on

### S7: the branch is checked out with changes

- Given a branch behind GitHub's, as in S1 or S2, checked out in a worktree with uncommitted changes
- When `apply` pushes it
- Then the checkout's HEAD, index and files are unchanged, the local branch stays where it was, nothing is pushed, and `apply` goes on

### S8: no `origin`

- Given a repository with no `origin` remote
- When `apply` pushes a branch there
- Then nothing is pushed and no branch moves, as today

## Constraints

- D42 holds: no checkout moves except the branch's own, and only when it is clean. No force push, ever.
- No extra network call: the decision uses the `origin/<branch>` ref the session's fetch left.

## Decisions

- The fix is in the one push function every `apply` push goes through (the engine's git writer), not in each caller, so run branches, unit branches and `record`'s push of a unit branch all get it.
- Compare the local branch with `origin/<branch>`: local is an ancestor of it (and differs) → bring local forward, no push; otherwise → push, and let GitHub refuse a diverged branch as today.
- Bringing the local branch forward uses the existing move that respects D42 (plumbing when not checked out, a fast-forward in a clean checkout). When its checkout has changes, leave the local branch behind and go on: everything `apply` reads already takes the freshest of local and GitHub's (D25), so nothing needs the local branch moved.

## Open questions

None
