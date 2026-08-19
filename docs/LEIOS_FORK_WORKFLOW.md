# Leios fork workflow (trial period)

Status: adopted 2026-08-18 for the opening week(s) of the six-week Leios
plan, while the work is trialed in William's fork before anything
touches the IntersectMBO repository.  Archive this file when the train
moves upstream.

## Why a fork

Carlos does not want unvetted, AI-assisted work landing in the official
issue tracker or repository, and he is away this week.  The trial
therefore runs entirely in `williamdemeo/formal-ledger-specifications`;
upstream sees nothing until each chunk has been human-reviewed.

## Local layout: two clones, two jobs

+  `~/git/IO/fls` (remotes: `origin` = IntersectMBO, `fork` = the fork)
   hosts the CODE TRAIN.  All chunk worktrees are created here, under
   `~/git/IO/fls/worktrees/`, so the house tooling applies unchanged:
   `link-worktrees.sh fls` links `.claude` into every worktree of this
   clone, and the project `CLAUDE.md` is found by upward search.  The
   push destination is chosen per push (`git push -u fork <branch>`).
+  `~/git/williamdemeo/fls` (remote: `origin` = the fork) hosts only
   the admin worktree `worktrees/1-leios-tracking-issue`.  Populate and
   roadmap edits happen there; plain `git push` goes to the fork.  The
   house tooling does not know this clone: if a Claude session is ever
   run there, symlink `~/git/IO/fls/CLAUDE.md` to
   `~/git/williamdemeo/fls/CLAUDE.md` and `~/git/IO/fls/.claude` into
   the worktree root by hand.

## Topology

+  `fork/leios-main` is a fast-forward mirror of `origin/leios-main`
   (origin = IntersectMBO).  Never commit to it directly.  Sync with:
   `git fetch origin && git push fork origin/leios-main:refs/heads/leios-main`.
+  `fork/1-leios-tracking-issue` is the admin branch, based on
   `leios-main`: the two roadmap files, this document, and the populate
   writebacks live here.  It is never merged upstream as-is.
+  One branch per plan issue, in a linear stack, each cut from the head
   of its predecessor:
   `leios-design-note` (M1-1) → `leios-abstract` (M1-2) →
   `leios-types` (M1-3) → `leios-pparams` (M1-4) →
   `leios-pool-key` (M1-5).  The top of the stack is always the fully
   integrated state; no separate integration branch is needed.

## The PR train

Each chunk becomes a draft pull request inside the fork (base:
`leios-main` for the bottom chunk, the predecessor branch above that),
so CI runs per chunk and each PR description is written once and reused
upstream.  Per-chunk ritual, from `~/git/IO/fls/master`:

    cd ~/git/IO/fls/master
    git worktree add -b leios-<chunk> ../worktrees/leios-<chunk> <base>
    ~/git/williamdemeo/claude-tooling/main/scripts/link-worktrees.sh fls
    cd ~/git/IO/fls/worktrees/leios-<chunk>
    # ... work, small commits, nix typecheck ...
    git push -u fork leios-<chunk>
    gh pr create --repo williamdemeo/formal-ledger-specifications \
      --head leios-<chunk> --base <leios-main-or-predecessor> --draft \
      --title ... --body-file ...

Always pass `--base` explicitly: the fork's default branch is `master`,
so an unbased PR targets the wrong branch.  Enable GitHub Actions on the
fork once (Settings → Actions); the pipeline triggers on PRs targeting
`leios-main`.  Mirror syncs also trigger push CI and the pipeline
creates `<branch>-artifacts` branches on the fork; both are harmless.
If review changes land in a lower chunk, rebase the stack upward and
force-push with lease; fork branches are ours to rewrite (check
`git cherry` first, per house rules).

Fork PRs are review artifacts and CI carriers, never merge targets: the
mirror stays pristine, the integrated state is the top of the stack, and
each fork PR simply closes when its upstream twin merges.

## Plan files and populate

Two copies of each roadmap exist on purpose:

+  Admin-branch copies (fork): the `**Repository**:` header points at
   `williamdemeo/formal-ledger-specifications`; populate runs against
   these and writes `(#N)` issue numbers back into them.
+  Pristine copies (the leios-main worktree, uncommitted): header points
   at IntersectMBO; these are the eventual upstream versions and must
   never accumulate fork issue numbers.

Populate runs ONCE, before the first chunk of work: issue numbers then
exist for commits and PR bodies to reference, and any surprise in the
engine surfaces before the week starts.  Refresh the admin copies from
the pristine worktree first, then point the header at the fork:

    cd ~/git/williamdemeo/fls/worktrees/1-leios-tracking-issue
    cp ~/git/IO/fls/worktrees/leios-main/docs/GITHUB_PROJECT_6WEEK.md docs/
    # edit the header:  **Repository**:  `williamdemeo/formal-ledger-specifications`

Populate and update run from that admin worktree, engine straight from
the checkout (stdlib-only Python; `gh` must be authenticated):

    ENGINE=~/git/williamdemeo/github-project/main/scripts
    python3 $ENGINE/gh_project_lint.py     docs/GITHUB_PROJECT_6WEEK.md
    python3 $ENGINE/gh_project_populate.py docs/GITHUB_PROJECT_6WEEK.md --dry-run
    python3 $ENGINE/gh_project_populate.py docs/GITHUB_PROJECT_6WEEK.md
    git add -u && git commit -m 'Record populated issue numbers' && git push fork
    python3 $ENGINE/gh_project_update.py   docs/GITHUB_PROJECT_6WEEK.md --check

Never run populate on a file whose header still says IntersectMBO, and
never run update before the first populate (it rebuilds the generated
regions from GitHub, which for an unpopulated plan means erasing every
authored issue body).

## Upstream transition (when Carlos is back)

Bottom of the stack first: push `leios-design-note` to origin, open the
PR against upstream `leios-main` reusing the fork PR's description, and
retarget each subsequent PR as its predecessor merges (the usual
stacked flow).  The roadmaps go upstream separately, from the pristine
copies, once the team adopts the plan; upstream populate happens only
then, and only with team agreement.

## Provenance

Every commit is authored by William; nontrivial AI-written changes end
the commit body with the AI-assistance line, per repo convention.  This
is what makes the trial defensible: nothing reaches upstream without a
human pass, and origins are declared.

## Orientation for future Claude Code sessions

The auto-memory entry `fls-leios-roadmap` carries current state.  The
plan of record for the trial is `docs/GITHUB_PROJECT_6WEEK.md` on the
admin branch; work proceeds chunk-by-chunk per its issues, in fork
worktrees, under the rules above.  Do not create issues or PRs on the
IntersectMBO repository; the fork is the only write target until
further notice.
