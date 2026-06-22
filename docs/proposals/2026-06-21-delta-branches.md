---
lep: 2026-06-21-delta-branches
title: Delta Branches
authors:
  - Mattias Jansson
status: Draft
created: 2026-06-21
updated: 2026-06-21
discussion: <TBD — open the discussion CR for mjansson/delta-branches>
replaces: 2026-05-03-modified-file-tracking
---

# Delta Branches

## Summary

This proposal lets a single Lore instance carry several ongoing streams of work in one working tree. It introduces the
**delta branch**: a lightweight branch that holds one stream as its own strictly linear line of history — a line that
reduces to a single net change — layered over the branch the developer is on.

A developer can **attach** a delta branch to materialize its changes in the working tree, **detach** it to park them
off-disk, keep several attached at once, let an editor or tool checkpoint work into one continuously, and move a delta
branch as a unit — rebasing it onto a different base, or committing it as a new revision on a chosen or new branch.

One mechanism thereby covers a wide range of needs that today are unmet or served only by separate, partial workarounds:
automatic personal backup of work, several work streams in flight at once, carrying work across branch switches,
toggling changes on and off to test combinations, keeping long-lived debug or utility changes switchable, and picking up
in-progress work on a developer's other machines.

Concretely, it folds three mechanisms Lore has today — dirty-file tracking, staging as the recording of commit intent,
and the personal backup branch already used in UEFN — into a single concept, and on that foundation adds what Lore has
never had: multiple parallel change sets in one working tree.

## Motivation

A developer working in a Lore repository rarely has just one thing in flight. Over a single sitting, one working tree
accumulates an urgent fix started on top of a half-finished feature, a broad rename touching dozens of files beside a
couple of unrelated one-line corrections, a debug line kept only for local troubleshooting, and a second attempt at a
function whose fate is undecided. These are distinct streams of work. Each is destined to become its own revision, on
its own schedule and often on its own branch — and some are never meant to be committed at all.

Lore gives a repository instance a single current branch and a single working tree, and every local modification in that
tree collapses into one undifferentiated set. The tree records *that* a file changed, not *which* stream the change
belongs to. The moment a developer is doing two things at once, that single set stops matching how they think about
their work, and the cost shows up in several ways:

1. **Concurrent streams can't be kept apart while in progress.** There is nowhere to say "these changes are the fix,
   those are the feature." The grouping lives only in the developer's head and must be reconstructed from memory at
   commit time. When two streams touch the *same* file — a feature edit and a debug print three lines apart — even a
   file-by-file sort can't separate them; the boundary the developer cares about is sometimes a hunk.

2. **Setting a stream aside means losing it or freezing everything.** To get one stream out of the way, a developer must
   either commit it onto the branch before it is ready, or switch branches — which evicts *every* other in-progress
   stream from the working tree at once. There is no way to park one stream while the others stay live and editable.

3. **Moving a stream to another branch is manual reconstruction.** Work often starts on the wrong base, turns out to
   belong on a new branch, or needs to land on a specific existing one. Redirecting an in-progress stream onto a
   different branch means re-deriving and re-applying its changes by hand, with no record of which changes made up the
   stream.

4. **In-progress work isn't captured continuously or durably, and is trapped on one machine.** Until a developer
   commits, a stream exists only as uncommitted working-tree state — vulnerable to loss and invisible to tooling. No
   editor or build tool can checkpoint the work of a given stream automatically as it happens, so the natural unit for
   incremental, durable backup simply doesn't exist — and what state there is stays on the one machine it was typed on,
   neither backed up off-machine nor reachable from the developer's other machines.

5. **The only real separation moves the work out of the tree.** Keeping streams genuinely apart today means separate
   branches or separate instances. Both separate work in *time* (you switch between them) or in *physical space on disk*
   (a second checkout to manage) — neither lets several streams stay co-resident, visible, and editable in the *same*
   tree at once, which is exactly where the work is happening. And each workaround is itself overhead to run and keep
   track of: branches to name and switch, instances to clone and manage, stashes to juggle.

6. **There is no way to toggle changes in and out.** Developers routinely want to test a combination of in-progress
   changes, switch one out to isolate a problem, or keep a debug print or local utility tweak around indefinitely
   without it ever shipping. Today a change is either present in the working tree or reverted; there is no durable,
   switchable on/off state for a set of changes.

The result is that developers either serialize naturally-parallel work, pay the overhead of extra branches and checkouts
to run it in parallel, or let everything pile into one tree and disentangle it by hand — re-deriving boundaries that
were obvious while the code was written and are easy to get wrong afterward.

The continuous-capture need is not hypothetical. Unreal Editor for Fortnite (UEFN) already relies on it today: the
editor auto-saves every change a creator makes and commits it onto a per-user backup branch kept hidden from the
creator, purely so that nothing is ever lost. That pattern works and is in production — but it is a *single*, hidden,
unmanaged branch. A creator cannot see it as several streams, organize in-progress work into separate units, park one
while keeping others live, or move a unit onto a different branch. It captures work durably while leaving every other
need above unmet.

This matters more as Lore moves into large monorepos, where duplicating a checkout per stream is expensive, and as
automated tooling and agents produce changes a human must later sort and attribute. The recent move to make per-file
modification state a first-class, persistently tracked concept
([`2026-05-03-modified-file-tracking`](2026-05-03-modified-file-tracking.md)) closed one part of this gap; the remaining
need is to let a single instance carry **several ongoing streams of work in one tree** — separable down to the hunk,
parkable without disturbing the others, movable between branches, and continuously and durably captured as work happens.

## Goals / Non-Goals

### Goals

1. **Carry several streams of work co-resident in one instance and one working tree.** A developer should keep multiple
   in-progress streams live at the same time without a second checkout and without switching branches. *(Motivation 1,
   5.)*

2. **Separate streams down to the hunk, not just the file.** Two changes in the same file should be assignable to
   different streams. *(Motivation 1.)*

3. **Park and restore a stream without disturbing the others.** A developer should remove one stream's changes from the
   working tree and bring them back later, with the other streams untouched and without committing the parked stream
   onto a branch prematurely. *(Motivation 2.)*

4. **Capture a stream's work continuously and durably, on every machine.** An editor or tool should be able to
   checkpoint the active stream automatically as work happens, producing durable history that survives a lost working
   tree; when remote-synced, that history is also an off-machine backup the developer can pick up on another of their
   machines. *(Motivation 4.)*

5. **Treat a stream as a movable unit.** A developer should rebase a whole stream onto a different base, and commit it
   onto a chosen existing branch or a new one, without re-applying its changes by hand. *(Motivation 3.)*

6. **Make managing many streams cheap.** Listing, attaching, detaching, and switching which stream is active should be
   low-overhead operations a developer runs routinely. *(Motivation 5.)*

7. **Toggle changes on and off.** A developer should switch a set of changes in and out of the working tree at will — to
   test combinations of in-progress work, isolate a problem, or keep long-lived debug and utility changes that never
   ship. *(Motivation 6.)*

### Non-Goals

- **Replacing regular branches or the remote review flow.** Delta branches are a working-state mechanism that feeds the
  existing branch/commit/push/change-request workflow, not a substitute for it.

- **Branching off a delta branch.** A delta branch is a single linear stream that reduces to one net change; nothing is
  ever based on it, and it never forks into sub-branches.

- **Merging to or from a delta branch.** A delta branch reaches a regular branch only by `commit` (squashing to its
  single net change); it is never a merge source or target.

- **Co-editing a delta branch.** A delta branch has a single writer, its creator (who works it from any of their own
  machines). A different user cannot write to it — at most they copy it onto their own ground as a new, locally-owned
  delta branch, a side benefit rather than collaboration. Genuine collaboration comes from committing the delta branch
  as a new full branch (or onto an existing branch) that others develop on, never from co-writing the delta branch
  itself.

- **Automatic semantic merging of overlapping streams.** This proposal defines what happens when two attached streams
  touch the same hunk; it does not promise conflict-free composition of arbitrarily overlapping edits.

- **A new content or diff format.** Delta branches reuse Lore revisions and the existing merkle tree; they do not
  introduce a separate patch representation.

## Proposed Design

This proposal introduces the **delta branch**: a lightweight branch that holds one stream of work as its own line of
history, layered over the branch the developer is on. Where a regular branch *represents a line of history*, a delta
branch *represents a single set of changes to apply* — recorded as a line of history internally, but understood and
manipulated as one delta.

**Ground and base.** The branch and revision the instance currently occupies is the *ground* — "where you are" in the
revision graph. Each delta branch records the base revision it was created from and grows independently of the ground.
(*Goal 1.*)

Visually, it is an ordinary multi-branch revision graph. You are grounded on one current branch; the delta branches
layer on top of that ground (the figure shows the v2 picture, with two delta branches attached at once):

```text
   main   ●────●────●────●   r4   ◄── ground: current branch (main), current revision (r4)
          r1   r2   r3   r4
                │    │
                │    └─▶──●──●──●  feature/login (another branch — forked at r3, not current)
                └─▶──●──●          release/1.0   (another branch — forked at r2, not current)

   the ATTACHED delta branches are applied on top of the ground (main @ r4) in the working tree:

          r4 ──┬──▶  ui-tweak     ●──●──●   (≡ one net change)   ◄── active
               └──▶  debug-logs   ●         (≡ one net change)

   DETACHED — recorded but parked off-disk (not in the working tree); each one is
   rebased onto the ground at the moment it is attached:

          bugfix        ●──●          (≡ one net change)
          experiment    ●──●──●──●    (≡ one net change)

   legend:  ●  a revision        ●──●──●  a delta branch's line of checkpoints
            ≡  the whole delta branch squashed to one net change; `commit` writes it
               as one new revision on a chosen or new branch
```

**The default delta branch.** Every instance always has one unnamed, default delta branch, attached and active unless
the developer makes another one active. Any change to a file that has not been assigned to a specific delta branch is
captured there, so by default *all* local work is already attributed to a delta branch. The developer opts into
organization by moving files (and, from v2, hunks) out of the default branch into separate delta branches — not by
opting in to tracking at all. (*Goals 1, 2, 6.*)

**No names.** A delta branch has no name; it is identified by an internal id — there is no name→id key as regular
branches have — and an optional `description` in its metadata carries a human-readable note of what it contains and
represents. The default delta branch is the active catch-all and carries no description; a developer describes a stream
rather than naming it. (*Goal 6.*)

**Presentation.** Delta branches are presented separately from regular branches: they do not appear in
`lore branch list`. A dedicated `lore delta-branch` listing shows them, each marked *attached* or *detached*, and which
one is the active branch. Keeping the two kinds in distinct views avoids blurring the regular-branch concept — a regular
branch is mutually exclusive in the working tree, a delta branch additive. (*Goal 6.*)

**This replaces dirty tracking and staging.** By default, local work is recorded as checkpoint commits on a delta branch
— by an editor or filesystem watcher, or by a CLI/API call that tells Lore of a change (the same call editors and
watchers use). A file's modification state becomes simply the presence of its changes in a delta branch rather than a
separate *dirty* flag, and choosing what goes into the next revision becomes *which delta branch you commit* rather than
a separate staged set. Both the per-file dirty flag from
[`2026-05-03-modified-file-tracking`](2026-05-03-modified-file-tracking.md) and the staged set collapse into
delta-branch membership, which is file-granular in v1 and hunk-granular from v2 (staging was file-only). `lore status`
reports each attached delta branch and the changes it holds, and `lore commit` commits a delta branch onto a branch.
This proposal supersedes that LEP, keeping its `--scan` reconciliation (see *Scanning*) while replacing the per-file
dirty flag. (*Motivation 4.*)

**The single-branch case is exactly today's behavior.** With only the default delta branch — the configuration a
developer who never splits work into streams stays in — the model reduces precisely to how Lore works today. The default
branch's membership is the dirty set; marking a file *intent-only* with `lore stage` is staging; `lore status` shows
exactly what it shows today; and `lore commit` squashes the default branch's net change into one revision — identical to
committing today's staged state. Delta branches are a strict superset: nothing about the single-stream workflow changes
until the developer creates a second delta branch — yet the unification and first-class automatic backups are already in
place underneath.

**Scanning.** Scanning becomes reconciliation of the filesystem against the attached delta branches: a change to a file
not in any delta branch is applied to the active delta branch (the default branch unless another is active), while a
change to a file already in a delta branch is applied to that branch. Where a file belongs to more than one attached
delta branch, attribution follows the same rule as interactive edits (open question 1).

**A line of history, handled as a unit.** A delta branch is a real sequence of revisions, not a single stored diff.
Incremental work accumulates as successive revisions on it, so the stream carries its own durable, inspectable history.
The developer never manages those revisions individually: `attach`, `detach`, `rebase`, and `commit` all operate on the
delta branch as a whole — on the net effect of its `base..latest` range. This is StGit's "a stack of commits handled as
one patch series" idea applied to a Lore branch. (*Goals 4, 5.*)

**Linear and non-branchable.** A delta branch is strictly linear: it has no internal branching, and nothing — neither a
regular branch nor another delta branch — can be based on it. Its checkpoint revisions exist only to record the stream's
progress; what a delta branch *means* is the single net change of its `base..latest` range, and that is what `commit`
ultimately delivers. Every delta branch is based directly on the ground rather than on another delta branch, so they are
peers over a common base, not a stack; attaching reconciles each against those already attached (see *attach* below).
(*Goals 2, 5.*)

**attach / detach.** `attach` materializes a delta branch's net change into the working tree; `detach` removes it,
leaving each file as the ground and the remaining attached delta branches define it. Several delta branches may be
attached at once (in v2), their changes overlaying in the one tree. Detaching parks a stream off-disk without committing
it onto the ground or evicting the others. (*Goals 1, 3.*)

**Attach is a rebase.** Attaching a delta branch updates its branch point to the current revision of the base branch and
replays its change there, on top of any already-attached delta branches — the equivalent of a git rebase onto the
ground. If the replayed change conflicts with the ground or with any other attached delta branch, the developer must
resolve the conflict before the attach completes, or abort and leave the branch detached. Lore's existing
conflict-resolution flow applies (diff3 markers and `resolve`/`abort`, as for a pending merge). Because every attach
reconciles conflicts up front, the set of attached delta branches is always mutually consistent in the working tree.
(*Goals 1, 3, 5.*)

**Ground advancement.** When the ground advances — a `sync`/`switch`, or a commit onto the current branch — Lore rebases
each attached delta branch onto the new ground, reusing the attach rebase and its resolve-or-abort flow per branch on
conflict. Detached branches are left untouched and rebase lazily the next time they are attached. (*Goals 1, 3.*)

**View filter.** In a sparse working tree, attach materializes only the branch's in-view paths; changes the branch
carries outside the current view stay recorded and commit in full, faulting onto disk only if the view later expands to
include them. Delta branches thus respect the sparse model rather than silently widening it. (*Goal 1.*)

**Active delta branch.** Exactly one attached delta branch is *active*. New modifications in the working tree, and
incremental checkpoints, flow into the active delta branch. Switching the active branch is a cheap pointer change once
several are attached (v2); under v1's single attached branch, switching streams is detach-then-attach. (*Goals 1, 6.*)

**Hunk-level membership (v2).** A change's membership in a delta branch is recorded at hunk granularity, so two edits in
the same file can belong to different streams. This needs no patch format: each branch stores the file's full content as
that branch sees it — the ground's file with only that branch's hunks applied — and composition is an ordinary 3-way
merge against the ground base, with hunk membership a presentation layer over those whole-file snapshots. This is a v2
capability; v1 tracks membership per whole file. How membership is presented and reassigned across attached branches is
refined in open question 1. (*Goal 2.*)

**Reassigning changes.** A developer or tool can move or copy a file's changes — or, from v2, an individual hunk's —
from one delta branch to another. This is how work is organized into streams after the fact: pull a stray edit out of
the default branch into the stream it belongs to, or copy a shared fix into a second stream that also needs it. (*Goals
2, 6.*)

**Reverting, not unstaging.** There is no unstage operation. To drop a change from a delta branch, a developer reverts
the affected file in that delta branch; because the branch keeps its line of history, the change can be resurrected
later by reverting to an earlier revision on the delta branch. Withdrawing a change is just another step on the stream,
not a separate staging state. (*Goals 4, 6.*)

**Incremental capture.** An editor, watcher, or build tool can request a checkpoint of the active stream; Lore records a
lightweight revision on the active delta branch. This gives continuous, durable backup per stream as work happens, with
no manual commit step. Their cadence and coalescing are an open question (below). (*Goal 4.*)

**Intent-only capture and large files.** Capture has two modes per file. By default a file's changes are recorded in
full as checkpoint revisions, giving the continuous durable backup above. A file can instead be marked *intent-only*
(via `lore stage`): the branch records that the file belongs to it and will be committed, but its content is captured
only at commit time, not on every checkpoint — exactly the classic staging behavior, now a capture mode rather than a
separate area. To avoid writing large amounts of data while iterating on a large file, an automatic policy records files
above a configurable per-repository size threshold (8 MiB by default) as intent-only. The trade-off is that an
intent-only file has no continuous backup trail between commits. (*Goals 4, 6.*)

**Unit operations: rebase, commit.** `rebase` reparents a delta branch's line of history onto a different base — a newer
revision on the current branch, or a revision on another branch — recomputing its net change against the new base.
`commit` squashes a delta branch's line of history into its single net change and writes that as a single new revision
on a chosen target branch — or on a new branch created from it. There is no partial commit at the primitive level:
`commit` always lands the branch's whole net change. Tooling can still present a partial commit, though — let the user
select what to commit and, behind the scenes, move the unselected changes onto a new delta branch before committing, so
the operation that runs is still a whole-branch commit. A delta branch is never a merge source or target: it reaches a
regular branch only by `commit`, never by merge. Together these let a developer move a stream between branches and
decide where and when it ships, treating the whole stream as one unit. (*Goals 3, 5.*)

**Local-only or remote-synced.** Each delta branch is configured as local-only or remote-synced. A local-only delta
branch never leaves the instance. A remote-synced delta branch sends its data and tracking to the server, so the
creator's in-progress work is backed up off-machine and available on the creator's other machines — a stream started on
a desktop can be picked up on a laptop. (*Goals 1, 4.*)

**Single writer.** A delta branch is owned by its creator, and only the creator updates it. The single writer is the
user, not a machine — the creator works the same remote-synced delta branch from any of their own machines, which is how
remote sync doubles as per-user backup and cross-machine pickup. No other user writes to it; this single ownership is
what lets a delta branch be freely rebased, moved, or split on the owner's machine: because no one else can advance it,
every rewrite is a purely local operation that never has to reconcile concurrent remote edits. A multi-writer delta
branch would instead force every rewrite to account for other users' changes — remote syncing, merging, and conflict
handling on each rebase or split — which is exactly the cost this constraint avoids. As a side benefit, a different user
can *copy* a remote-synced delta branch — handy for ad-hoc help like "copy my debug-logging branch to reproduce the
bug": Lore computes the source branch's net change, creates a new delta branch from the consuming user's current
revision, and applies that net change as its first revision, resolving any conflicts — the same rebase-onto-ground
operation as `attach`. The original stays the creator's; the consumer gets an independent, locally-owned copy on their
own ground. This is not a collaboration channel — real collaboration still goes through committing a delta branch as a
normal branch that others develop on. (*Goals 3, 5.*)

**Locks.** Delta branches interact with the exclusive locks from
[`2026-06-19-successor-locks-unmergeable-files`](2026-06-19-successor-locks-unmergeable-files.md). A lock on an
unmergeable file can be taken at either of two moments: when the file is first edited and tracked by a delta branch, or
only when that delta branch is committed onto a real branch. Taking it late lets a developer experiment locally without
holding a lock, accepting that the work may fail the lock's causal-safety check when it is committed; taking it early
reserves the file up front when the change is known to be real. The default policy is an open question. (*Goals 4, 7.*)

### Phasing

**v1 — one attached delta branch at a time.** The first cut materializes a single attached delta branch (the active
one); switching streams is detach-then-attach. Many delta branches still coexist in the instance — they are simply not
materialized simultaneously. In particular, today's separate staged state disappears in v1 — the default delta branch is
the single working set that `commit` writes, and `lore stage` no longer selects a commit subset but marks a file
intent-only. This already delivers the dirty-tracking-and-staging replacement, the default delta branch, continuous
capture and the intent-only/large-file policy, parking (`detach`), organization by whole-file reassignment and revert,
portability (`rebase`/`commit`), and local-or-synced single-writer sharing — Goals 3–6 and the full unification. Because
the working tree only ever reflects the ground plus one delta branch, edit attribution is unambiguous and the
composition and cross-branch-attribution problems (open question 1) do not arise. Stream membership is tracked per whole
file in v1; hunk-level membership (Goal 2) is deferred to v2.

**v2 — several attached delta branches overlaid.** The second phase materializes multiple delta branches in one tree at
once, completing Goal 1 (co-resident streams), adding hunk-level membership (Goal 2), and enabling Goal 7 (toggling
combinations on and off to test them). This is where open question 1 — composing overlapping branches and attributing
edits across them — must be answered; v1 is unaffected by it.

### Open design questions

The v2 multi-attach model must settle one design question. It is not a feasibility blocker — several attribution
semantics work — but it defines core v2 behavior, so it is a deliberate design task, not a tunable default (it also
appears under Unresolved Questions):

1. **Edit attribution across attached streams.** How edits the active delta branch makes to lines contributed by another
   attached delta branch are attributed, how a scan attributes a change to a file held by more than one attached delta
   branch, and how detaching or re-attaching either branch then behaves.

## Compatibility

- **Wire format** — Additive and backwards-compatible. Local-only delta branches never touch the wire. A remote-synced
  delta branch's latest pointer and revisions ride the existing branch path — its latest pointer reuses the normal
  branch key type for now — while its delta-branch metadata (marker, creator, local-only/synced flag, optional
  description) travels under a new dedicated key type keyed by id, with no name→id key. Branch enumeration keys off the
  name→id and id→metadata mappings, never the latest-pointer key, so a v(N-1) client — which has neither a name→id entry
  nor a recognized metadata entry for a delta branch — never discovers one; reusing the latest-pointer key type is
  invisible.
- **Client/server protocols** — Local-only delta branches involve no protocol. Remote-synced delta branches push and
  fetch like branches, with two constraints the server enforces: a delta branch is writable only by its creator, and
  another user consuming it receives the net change to seed a new, locally-owned delta branch rather than gaining write
  access. Attach/detach/active state stays local.
- **On-disk format** — Additive. A delta branch's latest pointer reuses the existing branch latest-pointer key type for
  now, but its metadata lives under a new dedicated mutable key type keyed by id (the delta-branch marker is implicit in
  the key type; creator, local-only/synced flag, optional description), with no name→id key. Branch enumeration keys off
  the name→id and id→metadata mappings, never the latest-pointer key, so delta branches never appear in branch listings,
  any number of them leaves regular-branch metadata untouched, and a branch's type is known from its key type without
  loading metadata. Per-instance state additionally records attach/detach status, the active pointer, and each file's
  membership (content-bearing or intent-only, no stored content until commit). The per-file *dirty* flag from
  [`2026-05-03-modified-file-tracking`](2026-05-03-modified-file-tracking.md) is replaced by delta-branch membership, so
  existing dirty state migrates into the default delta branch on upgrade. An upgraded Lore reads existing repositories;
  a downgraded Lore never discovers delta branches (no name→id entry, metadata under a key type it does not read), and
  the Migration Plan covers reconciling its work on the next upgrade.
- **CLI and public API** — New surface for creating, listing, attaching, detaching, setting the active stream, moving or
  copying changes between delta branches, rebasing, and committing. Delta branches surface through a dedicated
  `lore delta-branch …` command group, listed separately from `lore branch` and marked attached or detached.
  `lore status` now reports every attached delta branch and the changes it holds rather than reading dirty flags; the
  default delta branch's change set is equivalent to today's `lore status` output. `lore dirty` becomes a way to
  attribute a change to the default or a specific delta branch, and `--scan` reconciles the filesystem against the
  attached delta branches — routing each detected change to the delta branch that already holds the file, or, for a file
  in none, to the active branch. `lore commit` commits the default delta branch onto the current branch, and there is no
  `unstage`: a change is dropped by reverting the file in its delta branch and resurrected by reverting to an earlier
  revision on that branch. Staging collapses into assigning changes to delta branches, and `lore stage` marks a file
  *intent-only* — its content captured only at commit. Files above a per-repository size threshold (a repository config
  value, 8 MiB by default) default to intent-only automatically. Creating or configuring a delta branch chooses
  local-only or remote-synced; consuming another user's remote-synced delta branch creates a new local delta branch from
  the current revision.
- **Configuration** — adds one repository config value, the intent-only size threshold, defaulting to 8 MiB; an existing
  repository without it uses the default.

## Non-Functional Considerations

- **Concurrency** — Delta-branch write operations — checkpoint commits onto the active delta branch, changes to the
  active pointer, attach/detach, rebase, and commit — acquire the repository write token, gaining exclusive access and
  serializing against every other write operation, including those of other callers; reads such as `lore status` need no
  token. Several delta branches may be attached while a tool issues automatic checkpoints, but those checkpoints take
  the token in turn. Attach is a stateful, resolvable operation like a pending merge, so at most one attach is
  mid-resolution at a time.
- **Memory** — Materializing and overlaying net deltas must reuse Lore's streaming, sparse merkle model, streaming
  rather than buffering whole-tree state when attaching or detaching a stream.
- **Statelessness** — Introduces per-instance persistent state (the delta-branch set, attach state, and active pointer),
  held by the same machinery that backs today's anchor. No process-global state.
- **Determinism** — The net delta of a `base..latest` range, and the result of `commit`, must be deterministic for the
  same inputs, including when several streams are attached.

## Migration Plan

Two transitions apply even to the instance-local first cut. First, existing per-file dirty state and the staged set both
migrate into the default delta branch on upgrade — the separate staged state goes away, replaced by the default branch —
so uncommitted and staged local changes are preserved. Second, `lore status`, `lore dirty`, `lore stage`, and
`lore commit` change behavior as described in Compatibility, and `lore unstage` is removed in favor of reverting within
a delta branch. A downgraded Lore degrades gracefully: it never discovers delta branches — branch listing keys off the
name→id and id→metadata mappings, and delta branches sit outside both — so it operates only on regular branches and
cannot touch them. The staged-anchor structure (dirty and staged files) is retained on disk — only the new Lore stops
using it — so a downgraded Lore keeps tracking changes in it exactly as before. On the next upgrade, any nodes the older
client left in the anchor are folded into the default delta branch. Remote-synced delta branches add the new metadata
key type to the wire (additive); their latest pointer and revisions travel the normal branch path, but an older client
never discovers them and the server refuses pushes from non-creators, so they are safe. This transition is part of the
proposal, not deferred.

## Security Considerations

Local-only delta branches do not change the trust model — they never leave the instance, so no malicious peer or crafted
repository can observe or influence them beyond what is already possible for local working state. Remote-synced delta
branches send in-progress work and tracking to the server by the creator's per-branch opt-in. The server enforces
single-writer ownership, so a malicious peer cannot alter another user's delta branch; the most it can do with its own
is offer a net change that a consumer explicitly copies and resolves — no different in trust terms from consuming any
branch. The creator identity and sync flag on a delta branch become integrity-sensitive metadata.

## Privacy Considerations

Local-only delta branches expose no new data to other parties — checkpoints stay on the developer's machine.
Remote-synced delta branches, by the creator's opt-in, expose in-progress content, file paths, and the timing of
automatic checkpoints to the server — where they serve as the creator's backup and cross-machine store. In the
side-benefit case where a teammate copies one, that content reaches the teammate too — more than committed history
reveals, because work is captured continuously rather than at deliberate commits. Discarding a delta branch and
garbage-collecting removes it locally; whether a remote-synced delta branch can be fully expunged server-side is a
deletion concern to specify.

## Risks and Assumptions

**Assumptions**

- **Assumption:** developers want streams *co-resident* in one tree, not merely cheaper branch switching — *invalidated
  if:* users only ever work one stream at a time, in which case existing worktree-style instances already suffice.
- **Assumption:** conflicts between a delta branch and the ground or the attached set are infrequent, so resolving them
  at attach time is an acceptable cost — *invalidated if:* attaching routinely triggers conflict resolution, making
  co-resident streams burdensome to manage.

**Risks**

- **Risk:** reusing "branch" blurs the regular-branch concept, since a regular branch is mutually exclusive in the tree
  while a delta branch is additive — *mitigation:* a distinct mutable key type and a separate listing/command surface
  that presents delta branches apart from regular branches.
- **Risk:** automatic checkpointing floods the local store with short-lived revisions — *mitigation:* coalesce or
  garbage-collect checkpoint revisions, capture large files as intent-only rather than checkpointing their content, and
  squash on commit.

## Drawbacks

- Multiple simultaneously-materialized branches is a working-tree model no other Lore command assumes, so the change
  reaches staging, status, sync, and branch operations.
- Reusing "branch" means every branch operation must define its behavior for the delta-branch key type.
- Folding staging into the delta-branch model reworks the commit path — removing `unstage` and changing long-standing
  `stage`/`commit` behavior — which breaks scripts or integrations that rely on it.

## Alternatives Considered

### A standalone primitive (a named change-set distinct from branches)

Model each stream as a new first-class object — a "scene"/"stem"/changelist — separate from branches.

*Rejected because:* it would re-implement history, rebase, and durable storage that Lore branches already provide, and
it has no natural home for the continuous, durable backup trail that falls out of committing revisions onto a branch.

### Multiple instances over a shared store (worktree-style)

Use one instance per stream over a shared store, as Lore already supports.

*Rejected because:* each instance is a separate on-disk working tree, so streams are never co-resident; two streams
cannot overlay in the *same* file, and the developer pays a checkout and its management per stream.

### Selective staging at commit time

Keep one undifferentiated working set and separate streams by staging subsets just before each commit.

*Rejected because:* staging is file-granular and happens only at commit time; it preserves no grouping while work
continues, cannot park a stream, and offers no path to move a stream to another branch.

### A single git-stash-style stack

Add one anonymous stash stack to park work.

*Rejected because:* it is single and anonymous, not several named co-resident streams; it is not durable history, not
hunk-aware, and not portable across branches.

## Prior Art

- **Lore in Unreal Editor for Fortnite (the direct influence)** — Lore already backs UEFN with a single per-user backup
  branch the editor auto-commits every change onto (see Motivation). The continuous-capture model here comes from that
  practice; this proposal generalizes the one hidden branch into many managed, organizable, portable delta branches.
- **Jujutsu** — the working copy is itself a commit, auto-snapshotted on every command (no staging area), and anonymous
  heads let many lines of work coexist. Crucially for this proposal, jj materializes **one** working-copy commit per
  directory and runs parallel streams through `jj workspace` — separate directories, its git-worktree equivalent —
  rather than overlaying several changes co-resident in one tree, which is exactly the gap delta branches fill. jj also
  records conflicts as first-class objects and defers resolution, whereas delta-branch `attach` resolves or aborts up
  front. jj reaches continuous capture independently; it is convergent prior art, not the influence behind this
  proposal.
- **StGit / quilt** — maintain a stack of patches over a base and operate on the series as a unit (push/pop/refresh),
  the direct analog of attach/detach and unit rebase.
- **Mercurial** — `shelve` parks work, bookmarks are lightweight movable pointers, and topics/evolve provide mutable,
  named lines of in-progress history.
- **Git** — `stash` parks work but is single and anonymous; worktrees give parallel checkouts but in separate
  directories; branches are mutually exclusive in one tree. Together they illustrate the gap this proposal targets.
- **Sapling** — easy commit stacks and restacking, showing routine movement of a line of history onto a new base.
- **Perforce pending changelists** — the closest existing model for grouping in-flight work. A *pending changelist*
  groups opened files with a description in one workspace; files start in the *default changelist* and are moved into
  numbered ones (`p4 reopen`), and `p4 shelve` stores a changelist's files server-side so work is backed up and other
  users can `p4 unshelve` it — directly parallel to delta branches' default branch, organization-by-reassignment, and
  remote-synced sharing. It differs on the points this proposal needs: a file lives in exactly one changelist (no
  overlay of several streams in one tree), grouping is file-granular (not per-hunk), a changelist has no history of its
  own (no continuous per-stream backup trail), and it is tied to the workspace rather than being a portable, rebaseable
  unit.

The continuous-capture design here comes from Lore's own UEFN usage above.

Worth borrowing from elsewhere: StGit's stack-as-unit operations, and Perforce's default/numbered-changelist
organization plus shelve-to-share.

Worth avoiding: git stash's single, anonymous, opaque model.

## Unresolved Questions

These are deliberately deferred to the implementation phase and settled by experimentation; none is a feasibility
blocker — each has several workable answers. The first is the v2 edit-attribution design task above; the rest are policy
and default-behavior choices, each with a sensible starting default (e.g. coalesce checkpoints, suppress checkpoint
notifications, lock late).

- When the active delta branch edits a line contributed by another attached delta branch — or a scan detects a change to
  a file held by several attached branches — how is the change attributed, and how do detaching and re-attaching either
  branch then behave?
- After a delta branch is committed onto a branch, are its checkpoint revisions kept for local recovery or discarded
  immediately?
- Are a delta branch's checkpoint revisions visible in `lore history`, and how are they coalesced or garbage-collected?
- What checkpoint cadence and coalescing keep revision volume bounded — debounce/save-driven vs. per edit, and do
  consecutive checkpoints coalesce amend-style?
- Should a checkpoint commit on a delta branch suppress the notification fan-out that a regular branch advance triggers,
  so continuous checkpointing does not flood subscribers?
- Can a remote-synced delta branch be fully expunged server-side, or only discarded locally?
- Should an exclusive lock default to being taken when a file is first edited in a delta branch, or only when the branch
  is committed onto a real branch (see successor-locks LEP)?
