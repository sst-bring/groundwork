---
name: groundwork
description: Lay the groundwork for a build by distilling a lake of product and design research into a sourced decision register with owned open questions, then optionally rendering it into a repo's docs and AI context. Use when the user points at a pile of transcripts, briefs, decks, or prototypes and wants defensible architecture decisions out of it. Side-effecting and user-only: writes files to a path the user names. Produces a thoughts artifact (a decision register).
model: opus
allowed-tools: Bash, Read, Grep, Glob, Agent, Write, Edit, Skill, mcp__claude_ai_Notion__*, mcp__anytype__*
disable-model-invocation: true
---

# Groundwork

Turn a lake of product and design research into a reviewable decision register:
decisions stated as decisions, every driver carrying a locator, every gap
carrying an owner. Then, if asked, render that register into a repo's
documentation and AI context so the reasoning is applied on every future change
rather than filed.

```yaml
loads:
  - orchestration-runtime      # how to execute this block - read before anything
  - storage-backend            # where the register is saved
  - required-metadata          # schema fields + backend-specific title format
  - subagent-guide             # the agent catalog and the spawning rules
  - lake-provenance            # locators, extraction, source authority - binds every agent
  - templates/decision-register

constraints: [lake-provenance] # binds you and every agent you spawn

on-empty-invocation: |
  I'll distil a body of research into a sourced decision register.

  Please tell me:
  1. Where the research lives (`--lake <path>`), and whether any of it is a prototype
  2. The subject, if the lake covers more than one
  3. Optionally, where to render the documents afterwards (`--out <path>`)

  I'll inventory the sources, read them in parallel, and come back with claims
  and contradictions before proposing any decisions.

  Tip: `/groundwork --lake ~/research/atlas --out ~/code/atlas`

artifact:
  type: plan
  title-from: subject

delegation:                    # binds every sub-agent prompt this skill writes
  read-only: true              # no spawned agent writes, edits, or runs a mutating command
  scope: one cluster or one register per prompt, with explicit paths - never "look through the research"

orchestration:
  owns: [clustering, arbitration, synthesis, persistence, sync, user-dialogue, repo-writes]

  steps:
    - id: inventory
      inline: true
      produces: [sources, source-types, unreadable]
      reject: not flag(--lake)
      because: >
        Walk the lake and classify **every** file, not the ones whose extensions
        you recognise. Classify by how it will be read per
        `lake-provenance.md`'s extraction procedure, not by extension, because
        that is what determines the locator and because most unfamiliar formats
        turn out to be a zip or text with a header.

        Report the breakdown before going further: total count, the split by
        how each will be read, and the count of anything that reaches step 7 and
        is genuinely unreadable. A lake of 30 documents and a lake of 900 are
        different jobs, and "40 briefs plus 20 meeting recordings with no
        transcripts" is a different job again, one where the primary source is
        missing entirely. The user should hear that at the start rather than
        find it in a footnote at the end.

        No `--lake` means there is nothing to inventory, and guessing at a path
        is how you distil the wrong folder.

    - id: load-manifest
      requires: [inventory]
      when: exists(manifest)
      when-examples:
        match:    ["exists(manifest)"]
        no-match: ["not exists(manifest)"]
      inline: true
      produces: [unchanged-sources, new-or-changed-sources]
      because: >
        A prior run persisted a manifest of source identity, hash and what was
        extracted. Diff against it so a second pass reads what moved rather
        than the whole lake again. A tool that re-reads three hundred documents
        to revise one entry is a demo. Skipped on a first run, which is the
        whole of a cold pass.

    - id: cluster
      requires: [inventory, load-manifest]
      inline: true
      produces: clusters
      track-with: TodoWrite
      judgment: >
        Which sources cover the same subject, and where does each cluster's
        boundary fall? See "Clustering the lake" below.

    - id: prospect
      requires: [cluster]
      fanout: prospector
      over: clusters
      given:
        - { value: source-paths,      src: "the file list inventory produced, filtered to this cluster" }
        - { value: cluster-boundary,  src: "the cluster list cluster produced" }
        - { value: locator-rules,     src: "_thoughts/lake-provenance.md" }
      ask: [claims-with-locators, governing-commitments, contradictions, implicit-decisions, gaps]
      reject: not matches(source-paths, "/")
      because: >
        One prospector per cluster, all spawned in one message. Each returns
        sourced claims rather than a summary, which is what makes the register
        auditable later. The reject rule is the mechanical half of "never a
        directory": it catches "the research folder", not a real path list
        aimed at the wrong cluster, which stays your judgment in `cluster`.

    - id: map-prototype
      requires: [cluster]
      when: exists(lake.prototype)
      when-examples:
        match:    ["exists(lake.prototype)"]
        no-match: ["not exists(lake.prototype)"]
      agent: one-of [cartographer, codebase-pattern-finder]
      given:
        - { value: repo-root,         src: "the prototype path the user named" }
        - { value: exact-directories, src: "the prototype's own source tree, enumerated" }
        - { value: framing,           src: "this SKILL.md, section 'A prototype is a decision log'" }
      ask: [platform-choices-visible, where-each-is-visible, what-a-reader-would-have-to-accept]
      judgment: >
        Does this prototype need a whole-tree map, or one narrow lookup? See "A
        prototype is a decision log" below.
      because: >
        A prototype is code and both of these already read code, so do not
        write a third agent for it. Frame the ask as "what did this decide",
        not "how does this work". `one-of` rather than a hardcoded
        `cartographer` because the size of the prototype decides this and the
        block cannot know it: a forty-file Next.js app wants the map, a single
        route handler someone pasted in wants the narrow lookup, and spending
        a cartographer on the latter buys a context for one answer. Left in
        `unresolved[]` on purpose. Skipped when the lake has no prototype,
        which is common and costs nothing.

    - id: prior-context
      requires: [cluster]
      agent: archivist
      given:
        - { value: topic, src: "the subject, plus the cluster titles cluster produced" }
      ask: [prior-registers, prior-plans, prior-research, what-superseded-what]
      judgment: >
        Does this subject have a paper trail worth pulling? See "Prior context"
        below.
      because: >
        A register that re-decides something already decided is worse than no
        register. The archivist spans every storage backend and returns one
        briefing, and it runs concurrently with the prospectors so it costs no
        wall clock. This is also how a second pass on a related subject finds
        the first one.

    - id: read-cited
      requires: [prospect, map-prototype, prior-context]
      inline: true
      reject: matches(read-call, "limit|offset")
      because: >
        The prospectors name sources; the sources are the evidence. Read every
        document a load-bearing claim rests on, fully, in your own context. A
        sub-agent's citation is a pointer, not a verification, and you are
        about to put these claims in front of a stakeholder under your name. A
        partial read is how a non-goal in a closing section gets missed.

    - id: cross-check
      requires: [read-cited]
      inline: true
      produces: [cross-cluster-contradictions, commitment-conflicts, referenced-but-absent]
      judgment: >
        Which claims conflict once you hold every cluster at once, and which
        specific choices clash with a governing commitment declared elsewhere?
        See "Finding the contradictions nobody wrote down" below.
      because: >
        `owns:` names arbitration, and this is where it starts. A prospector
        sees one cluster and reports conflicts inside it; the conflicts that
        matter most span clusters, because the specific choice and the standard
        it violates are discussed by different people in different meetings.
        Nothing else in this run can see both halves. Skipping this step does
        not lose a nicety, it loses the class of finding the register exists to
        surface.

    - id: present-claims
      requires: [cross-check]
      inline: true
      presents:
        - source-count-read-and-unread
        - domain-nouns-and-relationships
        - contradictions-with-both-sides-and-locators
        - specific-choices-clashing-with-a-governing-commitment
        - governing-commitments-whose-content-is-absent-from-the-lake
        - claims-that-could-not-be-located
        - questions-the-lake-cannot-answer
      judgment: >
        Which gaps genuinely need a human, and which need another pass over the
        lake? See "Which questions are actually for the human" below.
      because: >
        Claims before decisions, always. This is the cheapest point at which
        the user can tell you that Dana left the company and her brief is
        stale, or that the deck you weighted heavily was never approved. Every
        decision downstream is written against what they say here.

    - id: verify-corrections
      requires: [present-claims]
      inline: true
      retry: { step: prospect, max: 1 }
      judgment: >
        Did the user's answer overturn a claim you asserted, and can you locate
        the correction in a source? See "A correction needs a locator too"
        below.

    - id: deeper-prospect
      requires: [verify-corrections]
      when: count(follow-up-clusters) > 0
      when-examples:
        match:    ["count(follow-up-clusters) > 0"]
        no-match: ["count(follow-up-clusters) == 0"]
      fanout: prospector
      over: follow-up-clusters
      track-with: TodoWrite
      given:
        - { value: source-paths,           src: "sources the first round surfaced but did not read, plus anything the user named" }
        - { value: cluster-boundary,       src: "the specific gap this round is aimed at" }
        - { value: what-round-one-left-open, src: "the gaps: sections of the first round's reports" }
        - { value: locator-rules,          src: "_thoughts/lake-provenance.md" }
      ask: [claims-with-locators, governing-commitments, contradictions, implicit-decisions, gaps]
      reject: not matches(source-paths, "/")
      because: >
        A second round aimed at what round one surfaced and what the user
        corrected, not a repeat of round one. The `when:` is the mechanical
        half of "do not invent follow-up work": if round one left nothing open
        and the user corrected nothing, `follow-up-clusters` is empty and the
        step skips, rather than handing you a fan-out over nothing and trusting
        you to decline it. Per the runtime doc, if you can say what you would
        check, it is a `when:` and not a `judgment:`.

    - id: reconcile
      requires: [verify-corrections, deeper-prospect, cross-check]
      inline: true
      produces: [decisions, overruled, prototype-choices]
      judgment: >
        Which source wins each contradiction, and what does that overrule? See
        "Source authority is applied, not decided" below. Re-run the cross-check
        over anything `deeper-prospect` added first: a second round brings new
        claims and new commitments, and a conflict between round one and round
        two is the same class of finding as one within round one.

    - id: present-decisions
      requires: [reconcile]
      inline: true
      presents:
        - decisions-with-drivers-and-locators
        - alternatives-and-why-not
        - prototype-choices-to-accept-or-override
        - open-questions-with-proposed-owners
        - what-was-overruled-and-under-which-rule
      judgment: >
        Which of these are decisions the research supports, and which are you
        about to invent? See "The line between derived and invented" below.
      because: >
        The user assigns the owners here, because you cannot. Most gaps belong
        to the PM who wrote the brief or the researcher who ran the study, and
        a register full of unowned questions is a document nobody actions.

    - id: register
      requires: [present-decisions]
      inline: true
      produces: register-body
      because: >
        Build the register per `templates/decision-register`, in your own
        context. This is not delegated: it is the synthesis, and `owns:` says
        so. Every entry states a decision rather than a topic, every driver
        carries a locator, every open question carries an owner, and the
        contradictions table names the losers.

    - id: challenge
      requires: [register]
      agent: assayer
      retry: { step: register, max: 1 }
      given:
        - { value: register-body,  src: "the register you built in register" }
        - { value: locator-rules,  src: "_thoughts/lake-provenance.md" }
        - { value: lake-root,      src: "the --lake path, so it can open what the register cites" }
      ask:
        - verdict
        - unsourced-drivers
        - resolutions-that-are-really-guesses
        - entries-contradicting-each-other
        - assumptions-with-no-invalidation-trigger
        - decisions-the-research-implies-but-nobody-wrote-down
        - prototype-choices-inherited-rather-than-surfaced
        - overruled-sources-dropped-silently
      on:
        "verdict: ship":   proceed-to-approve-register
        "verdict: revise": fix-the-findings-you-accept-rebuild-the-affected-entries-then-re-run-the-assayer-once
        "verdict: reject": take-the-findings-back-to-the-user-and-do-not-save
      judgment: >
        Is each finding right? See "The arbiter, not a relay" below.
      because: >
        The challenge happens before the user reads the register. A register
        they have already read is one they have already started trusting, and
        an invented driver caught afterwards costs you the whole document's
        credibility rather than one entry's.

    - id: approve-register
      requires: [challenge]
      inline: true
      produces: user-approval
      asks-user: "Shall I save this register with [N] decisions and [M] open questions?"
      because: >
        Unconditional. No `when:`, because there is no state of the world in
        which the register is saved unreviewed, and no `agent:`, because a gate
        is never delegated. `asks-user:` is exactly the question: the register
        was written out by `present-decisions` directly above, so restating any
        of it here duplicates what is on screen.

    - id: save
      requires: [approve-register]
      inline: true
      reject: not exists(user-approval)
      given:
        - { value: date-iso,   src: "date -Iseconds" }
        - { value: author,     src: "hyprlayer thoughts config --json, else git config user.name" }
        - { value: backend,    src: "hyprlayer storage info --json" }
        - { value: project,    src: "the subject, or the --out repo name if there is one" }
      title-format:
        git:      kebab-case-dated-slug     # 2026-08-25-atlas-decision-register
        obsidian: kebab-case-dated-slug
        notion:   human-readable-heading    # Atlas decision register
        anytype:  human-readable-heading
      destination:
        git:      thoughts/shared/plans/<title>.md
        obsidian: thoughts/shared/plans/<title>.md
        notion:   database-row (every required property populated, narrative as body)
        anytype:  object (every required property populated, narrative as body)
      because: >
        `status: draft`, `type: plan`. A register of proposed decisions is a
        plan, which is honest and needs no schema change. Note the deliberate
        departure from `create_plan`, which carries `reject: exists(open-question)`:
        this artifact is saved **with** its open questions, because owned
        questions are the deliverable rather than a defect. Save the manifest
        alongside it as `type: note`, related to the register, so the next run
        can diff instead of re-reading the lake.

    - id: sync
      requires: [save]
      when: backend == git
      when-examples:
        match:    ["backend == git"]
        no-match: ["backend == notion", "backend == anytype", "backend == obsidian"]
      inline: true
      run: hyprlayer thoughts sync
      because: >
        Pushes the register and manifest upstream. The other backends have no
        push/pull cycle in hyprlayer, so there is nothing to sync there.

    - id: derive-docs
      requires: [save]
      when: flag(--out)
      when-examples:
        match:    ["flag(--out)"]
        no-match: ["not flag(--out)"]
      inline: true
      produces: doc-set
      judgment: >
        What does the register support writing, and what would you be
        inventing? See "What the documents may contain" below.
      because: >
        Skipped entirely when the user wants only the register, which is a
        legitimate and common way to run this. The register is the reviewable
        artifact; the documents are one consumer of it.

    - id: present-docs
      requires: [derive-docs]
      when: flag(--out)
      when-examples:
        match:    ["flag(--out)"]
        no-match: ["not flag(--out)"]
      inline: true
      presents:
        - every-path-to-be-written
        - the-register-entries-each-document-rests-on
        - which-files-already-exist-and-would-be-overwritten
      because: >
        Every path, before anything is created. The user is reviewing whether
        the documents claim more than the research supports, and they cannot do
        that from a summary.

    - id: approve-docs
      requires: [present-docs]
      when: flag(--out)
      when-examples:
        match:    ["flag(--out)"]
        no-match: ["not flag(--out)"]
      inline: true
      produces: docs-approval
      asks-user: "Shall I write these [N] files into [path]?"
      because: >
        A second unconditional gate, because this one leaves files on disk
        outside the thoughts store. Approving the register is not approving the
        documents.

    - id: write-docs
      requires: [approve-docs, derive-docs]
      when: flag(--out)
      when-examples:
        match:    ["flag(--out)"]
        no-match: ["not flag(--out)"]
      inline: true
      reject: >
        not exists(docs-approval)
        or not flag(--out)
        or exit0(test -e <out>/docs/decisions.md)
      run: [Write per path in doc-set]
      because: >
        The `reject:` re-checks its own preconditions rather than trusting a
        satisfied `requires:`. A satisfied `requires:` proves the step ran, not
        that its check passed, and this step writes to the user's filesystem.
        The `exit0` clause stops a second run silently overwriting a register
        someone has since edited by hand: if it fires, present the diff and ask.

    - id: show-result
      requires: [save, write-docs]
      inline: true
      presents: [tree-listing-of-what-was-written, register-location, open-questions-still-unowned]
      because: >
        The plan was a promise and the tree listing is the evidence it was
        kept. Repeat the unowned open questions here, because they are the work
        the user leaves with.

        `requires:` names `save` as well as `write-docs`, and that is load
        bearing rather than redundant. A skipped step counts as satisfied for
        its dependents, so on a register-only run the whole document tail is
        satisfied the moment `derive-docs` skips. Anchoring on `save` is what
        keeps this step after the register exists instead of scheduling it in
        wave one. Every step in that tail carries its own `when: flag(--out)`
        for the same reason: guarding only `derive-docs` skips the derivation
        and then cheerfully writes the files anyway.

    - id: iterate
      requires: [show-result]
      inline: true
      accepts: [answered-open-questions, overturned-decisions, new-sources, reassigned-owners]
      re-challenges:
        step: challenge
        after: [decision-changed, driver-added, open-question-answered, source-added]
        bound: none                # every qualifying round, not a budget
      because: >
        Answering an open question usually unblocks a decision, and a decision
        that changes has to face the assayer again. This is the loop, and it is
        unbounded on purpose: a register is a document you circulate and
        revise, not a one-shot output. New sources re-enter at `inventory`, and
        the manifest is what keeps that cheap.
```

## Clustering the lake

You own this and it is the decision that most affects what the prospectors can
do. A cluster is a set of sources that argue about the same thing. It is not a
folder, not a date range, and not a document type.

Cluster by subject, because contradictions only surface inside a cluster. Two
sources that disagree about the batch window have to reach the same prospector,
or nobody notices they disagree. If you cluster by folder and the brief lives in
`docs/` while the transcript lives in `meetings/`, the disagreement survives
into the register as two confident entries.

Err toward fewer, larger clusters. A prospector with forty documents returns
fewer claims per document than one with eight, but a claim it returns is
reconciled against its neighbours already. Overlap is worse than size: the same
source in two clusters produces the same claim twice with different framings,
and you cannot tell downstream whether that is corroboration or duplication.

What failure looks like: a register where two entries cite the same transcript
for opposite conclusions, and neither mentions the other.

## A prototype is a decision log

When you frame the `cartographer` prompt in `map-prototype`, ask what the code
decided, not how it works.

A vibe-coded prototype has already settled auth, data layer, state management,
transport and deployment shape. Nobody reviewed any of it. A normal
cartographer prompt returns an accurate map of those choices, which is exactly
the wrong output, because an accurate map reads as a specification.

So ask for: what platform choices are visible in this tree, where each is
visible, and what a reader would have to accept if they kept it. Then every one
of those becomes an accept-or-override row in the register, never a driver.

What failure looks like: the register cites `src/api/pull.ts:41` as a driver for
"requests are pull-based." The prototype is not evidence of a decision. It is
evidence that somebody typed something on a Tuesday.

### Which agent

`agent: one-of [cartographer, codebase-pattern-finder]`, and the size of the
prototype decides it.

Take the `cartographer` when the prototype is a real application: several
directories, its own dependency set, more platform choices than you can count
by reading one file. You want the map, because the implicit decisions are spread
across it and any one file understates them.

Take `codebase-pattern-finder` when the prototype is one route handler, one
component, or a single script somebody pasted into the lake. A cartographer on
that returns a map of forty lines and costs a whole context, and the narrow
agent answers the only question you had.

When it is genuinely borderline, take the cartographer. Under-reading a
prototype means inheriting its decisions silently, which is the failure this
whole section exists to prevent, and over-reading one costs a context.

## Prior context

Skip the archivist only when you are certain there is no trail: a genuinely new
subject, or a lake you have already distilled in this session.

Otherwise pull it, because the cost is asymmetric. The archivist runs alongside
the prospectors and costs no wall clock, and what it finds changes the register's
job. A prior register on the same subject means this run is an update, so
entries that already exist keep their ids and the new work is the diff. A
superseded plan means a decision was made and then reversed, and a register that
re-proposes the reversed option will be read as not having done the reading.

What comes back is a briefing, not a verdict. A prior decision is a claim like
any other and needs a locator, so cite the artifact rather than asserting the
decision on the archivist's word.

## Finding the contradictions nobody wrote down

The prospectors hand you conflicts they found inside one cluster. Those are the
cheap ones. This step is for the conflicts that only exist when you put two
clusters side by side, and `lake-provenance.md` has the taxonomy of all six
kinds. Work it deliberately rather than hoping a conflict jumps out.

**Start from the governing commitments, not from the claims.** Every prospector
returned a list of them: references to a design system, a set of platform
tenets, a compliance regime, a prior ADR. For each one, sweep every cluster's
claims and ask which specific choices fall inside its remit. That is the search
that finds "all buttons should be blue" against "this follows the Acme design
system," and note that it does not work in the other direction. Reading the blue
button claim tells you nothing, because nothing about it looks wrong. The
commitment is what tells you where to look.

The reason to do it this way round is that the specific choice is the one that
gets built. Somebody writes the blue button on Monday. The commitment sits in a
document nobody reopens, and the conflict surfaces at design review six weeks
later, when it costs a sprint instead of a sentence.

**Then sweep for the other cross-cluster kinds.** Two numbers for the same
quantity from different meetings. Something in one document's non-goals that
another treats as a requirement. Two requirements that are each sensible and
jointly impossible, like a real-time dashboard against a nightly batch load. A
document that says one thing where the prototype silently does another.

**Most of these resolve to an open question, and that is the right answer.** You
cannot settle "buttons blue" against a design system that is not in the lake.
The entry is a question phrased as the decision somebody has to make, with the
owner named, and both sides carrying locators. Picking a side here is the single
worst thing this skill can do, because a resolved-looking entry stops anybody
from checking. Picking blue because a human said it out loud violates the
source-authority order outright: a transcript is evidence of discussion, and a
stated commitment to a standard outranks it. Picking the design system is just
as wrong, since you have not read it and it may say blue.

**Name every referenced-but-absent source explicitly**, even where you found no
conflict against it. A commitment whose content is not in the lake is the
likeliest hiding place for a contradiction you did not find, and listing it tells
the reader which document to go and fetch. It also stops the register reading as
though it checked something it could not.

What failure looks like: the register asserts buttons are blue, cites the
transcript, and never mentions the design system. Six weeks later somebody opens
the design system and the register's credibility goes with it.

## Which questions are actually for the human

At `present-claims` you will have more open questions than are worth asking.
Split them.

A question another pass over the lake can answer is not a question for the
user. Asking it spends their attention on work you could have done, and it
teaches them that this tool needs supervision. Send those to `deeper-prospect`.

A question for the user is one where the lake genuinely has no answer: which of
two contradicting sources is authoritative, whether a document was ever signed
off, who owns a gap, whether a stated constraint still holds. These turn on
facts about the organisation rather than facts in the documents.

The asymmetry that decides borderline cases: a question you should have
researched costs the user thirty seconds of irritation. A question you guessed
at instead of asking costs a wrong decision in a register that looks sourced.
Ask when unsure.

## A correction needs a locator too

When the user corrects a claim, you have two things that disagree: a locator and
an assertion. The locator is not automatically wrong.

Usually the correction is right and the source is stale, in which case find what
makes it stale and cite that instead. Sometimes the user is remembering a
conversation that never made it into the lake, and then the correction is a new
claim with the user as its locator, which is legitimate and must be recorded as
such rather than laundered into a document citation.

Occasionally the user is simply wrong about their own project, and the source
says what you said it said. Say so, quote it, and let them decide. That is
uncomfortable and it is the job: a register that silently adopts a
misremembering is worse than one that surfaces the disagreement.

Where the correction invalidates a whole cluster's reading, `retry:` back to
`prospect` rather than patching entries by hand. One re-run, and if it is still
wrong the problem is your clustering.

What failure looks like: the register asserts something with a page citation,
and the cited page says the opposite, because you took a correction and kept the
old locator.

## Source authority is applied, not decided

`lake-provenance.md` carries the order. Your job at `reconcile` is to apply it
and record the application, not to weigh each contradiction on its merits.

This is deliberate. Per-contradiction judgment feels more careful and is less
defensible: you will unconsciously favour whichever source supports the cleaner
architecture, and nobody reviewing the register can see you doing it. A declared
order applied consistently is auditable even where it is wrong.

Two rules carry most of the weight. A transcript is evidence of discussion, not
decision, unless someone states a decision as made. And later beats earlier only
at equal document type, so a recent hallway conversation does not overturn a
signed-off brief.

Record every application in the contradictions table: both sides, both
locators, the winner, the rule. Never drop the loser. The overruled claim is
frequently the one a stakeholder remembers hearing, and a register that does not
mention it reads as though it missed the meeting.

Where the order genuinely does not resolve a contradiction, that is an open
question, not a coin flip.

## The line between derived and invented

The register may assert what the research supports. It may not assert what the
research implies to you.

Design research reliably yields domain nouns and their relationships, actors and
their capabilities, workflows and states, constraints stated as numbers,
explicit non-goals, and named integration surfaces. Those are extractions and
they belong in the register with locators.

It does not yield language, framework, cloud service, database engine, service
boundaries, transport mechanism, or deployment topology. When one of those
appears in a source it is an aside by whoever was in the room, and it enters the
register as an implicit decision to be reviewed, never as a driver.

The failure mode is asymmetric, and that asymmetry is the whole reason for the
rule. A wrong extraction is visible: a reader sees "requests are pull-based" and
says no, we decided the opposite. A wrong inference is invisible, because
"therefore Service Bus" is a sentence nobody can check against a pile of
transcripts. Only one of those two errors survives review.

So when an entry's resolution names a technology, treat it as a defect until you
can point at the locator where a person chose it.

## The arbiter, not a relay

The assayer's report is input, not truth. You decide which findings to accept,
and you state the call and your reason.

Accept anything where it opened a locator and found something else there. That
is a checkable fact and it outranks your memory of writing the entry.

Push back where it flagged honest uncertainty as a defect. An entry that records
an assumption plus its invalidation trigger is a correct entry, and an assayer
that wants every entry decided has misread the artifact. Say so rather than
resolving the entry to satisfy it.

Take a `verdict: reject` seriously and check it yourself before acting. A reject
sends the pass back to the lake, which is expensive, and it is the finding most
likely to be an over-reaction to a handful of bad entries. If three entries are
wrong, that is a revise.

What failure looks like: you relay all nine findings to the user as though they
were nine facts, and they now have to arbitrate a review they did not read. That
is your job, and `owns: [arbitration]` says so.

## What the documents may contain

Six files, and the constraint is content rather than confidence.

`docs/domain-model.md` carries the glossary, entity relationships, state
machines and actors, each with a locator. This is the highest-confidence
extraction in the pass and the document that justifies the whole exercise.

`docs/constraints.md` carries the falsifiable statements: numbers, non-goals,
named integrations, compliance requirements, each with a locator and an
invalidation trigger. Short and dense. This is the document people argue with
productively.

`docs/technical-design.md` is deliberately mostly empty, and that is a feature.
Real headings for the decisions that must be made, and under each one the
constraints that bound it plus the open questions blocking it. So a Transport
section does not say "Azure Service Bus." It says pull-based per D3,
fifteen-minute window per C7, at-least-once versus exactly-once open and owned
by Dana. Architecture homework, pre-researched.

`docs/open-questions.md` carries the table with owners, verbatim from the
register.

Plus `CLAUDE.md`, kept thin: the domain vocabulary and the non-goals. Those are
the two things an agent gets wrong constantly and each costs a paragraph. And
`docs/decisions.md`, the flat "we chose X because [locator]" list, which
recovers the durable-why value cheaply and can be expanded into numbered ADRs
later by anyone who wants them.

What is not written, ever, unless a register entry names it as a decided
decision with a locator: dependency manifests, lockfiles, CI configuration,
Dockerfiles, language or framework choice, and directory skeletons. A directory
layout is a technical decision wearing a filesystem costume.

That constraint is what makes the derived-versus-conventional labelling
unnecessary. Everything written traces to an entry by construction, so there is
no conventional-default column to review. A label reading "conventional" is an
honest way of saying "I made this up," and a user skimming forty paths will
approve it, because reviewing forty paths for unstated assumptions is precisely
the work this tool was supposed to do for them.
