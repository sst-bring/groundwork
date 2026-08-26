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
  - plain-writing              # how everything this skill writes must read
  - templates/decision-register

constraints: [lake-provenance, plain-writing]   # bind you and every agent you spawn

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
      presents: [total-count, split-by-how-each-will-be-read, count-genuinely-unreadable]
      because: >
        Classify by how a file will be read per `lake-provenance.md`, never by
        its extension: that is what determines the locator, and most unfamiliar
        formats turn out to be a zip or text with a header. "40 briefs plus 20
        recordings with no transcripts" is a different job from "60 documents",
        and the user hears it now rather than in a footnote.

    - id: load-manifest
      requires: [inventory]
      when: exists(manifest)
      when-examples:
        match:    ["exists(manifest)"]
        no-match: ["not exists(manifest)"]
      inline: true
      produces: [unchanged-sources, new-or-changed-sources]
      because: >
        A prior run persisted source identity, hash and what was extracted.
        Diff against it so a second pass reads only what moved. A tool that
        re-reads three hundred documents to revise one entry is a demo.

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
        - { value: writing-rules,     src: "_thoughts/plain-writing.md" }
      ask: [buildable-findings, governing-commitments, contradictions, implicit-decisions, gaps]
      reject: not matches(source-paths, "/")
      because: >
        One prospector per cluster, all in one message. `lake-provenance.md`'s
        build test binds them, so what comes back is findings that change an
        artefact, not a summary of what people said. Expect most of the lake to
        be discarded. The `reject:` catches "the research folder"; a real path
        list aimed at the wrong cluster stays your judgment in `cluster`.

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
        not "how does this work": an accurate map of a prototype reads as a
        specification, and nobody reviewed those choices.

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
        register. The archivist spans every backend and returns one briefing,
        and it runs beside the prospectors so it costs no wall clock.

    - id: read-cited
      requires: [prospect, map-prototype, prior-context]
      inline: true
      reject: matches(read-call, "limit|offset")
      because: >
        A sub-agent's citation is a pointer, not a verification, and you are
        about to put these claims in front of a stakeholder under your name.
        Read every document a load-bearing claim rests on, fully. A partial
        read is how a non-goal in a closing section gets missed.

    - id: cross-check
      requires: [read-cited]
      inline: true
      produces: [cross-cluster-contradictions, commitment-conflicts, referenced-but-absent]
      judgment: >
        Which claims conflict once you hold every cluster at once, and which
        specific choices clash with a governing commitment declared elsewhere?
        See "Hunting the quiet contradictions" below.
      because: >
        `owns:` names arbitration and this is where it starts. A prospector
        sees one cluster; the conflicts that matter most span clusters, because
        the specific choice and the standard it violates get discussed by
        different people in different meetings. Nothing else in this run can
        see both halves.

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
        Claims before decisions, always. The cheapest point at which the user
        can tell you a brief is stale or a deck was never approved, and every
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
        - { value: writing-rules,          src: "_thoughts/plain-writing.md" }
      ask: [buildable-findings, governing-commitments, contradictions, implicit-decisions, gaps]
      reject: not matches(source-paths, "/")
      because: >
        A second round aimed at what round one surfaced and what the user
        corrected, not a repeat of round one. The `when:` is the mechanical
        half of "do not invent follow-up work".

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
      reject: not matches(entry, "frontend|backend|infrastructure|bounds")
      because: >
        The synthesis, and `owns:` says it is never delegated. Build it per
        `templates/decision-register`: every entry a decision rather than a
        topic, every driver carrying a locator, every question an owner. The
        `reject:` is the mechanical backstop for the build test: an entry that
        names no artefact it changes, and does not bound one that does, is cut
        rather than written down. See "The build test is the whole filter".
        `plain-writing.md` governs how it reads: decision first, plain words,
        active voice, no preamble and no padding.

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
        This happens before the user reads the register, because one they have
        already read is one they have already started trusting. An invented
        driver caught afterwards costs the whole document's credibility rather
        than one entry's.

    - id: approve-register
      requires: [challenge]
      inline: true
      produces: user-approval
      asks-user: "Shall I save this register with [N] decisions and [M] open questions?"
      because: >
        Unconditional: there is no state of the world in which the register is
        saved unreviewed, and a gate is never delegated. `asks-user:` is
        exactly the question, since `present-decisions` put the register on
        screen directly above.

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
        `status: draft`, `type: plan`: a register of proposed decisions is a
        plan, so no schema change is needed. Deliberately unlike `create_plan`,
        which refuses to save a plan carrying an open question, this one saves
        **with** them, because owned questions are the deliverable. The
        manifest saves alongside as `type: note`, related, so the next run can
        diff.

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
        push/pull cycle, so there is nothing to sync.

    - id: derive-docs
      requires: [save]
      when: flag(--out)
      when-examples:
        match:    ["flag(--out)"]
        no-match: ["not flag(--out)"]
      inline: true
      produces: doc-set
      documents:                    # exhaustive: nothing outside this list is written
        "docs/domain-model.md":     "entities, relationships and state machines, each with a locator, plus a mermaid erDiagram and stateDiagram rendered from the cited rows only"
        "docs/constraints.md":      "numbers, non-goals, integrations, each with a locator and an invalidation trigger"
        "docs/technical-design.md": "headings for the decisions still to make, each with the constraints bounding it and the questions blocking it"
        "docs/open-questions.md":   "the owner table, verbatim from the register"
        "docs/decisions.md":        "flat we-chose-X-because-locator lines"
        "CLAUDE.md":                "domain vocabulary and non-goals only"
      never-writes: [dependency-manifest, lockfile, ci-config, dockerfile, language-choice, directory-skeleton]
      judgment: >
        What does the register support writing, and what would you be
        inventing? See "What the documents may contain" below.
      because: >
        The register is the reviewable artifact; the documents are one
        consumer of it. Skipped when the user wants only the register.

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
        A second gate, because this one leaves files on disk outside the
        thoughts store. Approving the register is not approving the documents.

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
        satisfied `requires:`, which proves a step ran and not that its check
        passed. The `exit0` clause stops a second run overwriting docs someone
        has edited by hand: if it fires, present the diff and ask.

    - id: show-result
      requires: [save, write-docs]
      inline: true
      presents: [tree-listing-of-what-was-written, register-location, open-questions-still-unowned]
      because: >
        The plan was a promise and the tree listing is the evidence it was
        kept. Repeat the unowned open questions, because they are the work the
        user leaves with. `requires:` names `save` as well as `write-docs`
        deliberately; the README's "What it caught" explains why a skipped step
        needs an anchor.

    - id: iterate
      requires: [show-result]
      inline: true
      accepts: [answered-open-questions, overturned-decisions, new-sources, reassigned-owners]
      re-challenges:
        step: challenge
        after: [decision-changed, driver-added, open-question-answered, source-added]
        bound: none                # every qualifying round, not a budget
      because: >
        Answering an open question usually unblocks a decision, and a changed
        decision faces the assayer again. Unbounded on purpose: a register is
        circulated and revised, not one-shot. New sources re-enter at
        `inventory`, and the manifest keeps that cheap.
```

## Judgment

**The build test is the whole filter.** `lake-provenance.md` carries it and it
decides more of the output than every other rule combined. An entry earns its
place by naming the artefact it changes: a schema, an endpoint, a component, a
screen state, a pipeline, a deployment target. A non-code fact that bounds one of
those survives attached to it, never alone. Everything else goes, however well
sourced. Who said a thing, when, and whether two people talked past each other is
not this document's business, and a correctly quoted, precisely located claim that
changes no artefact is still noise. Expect to discard most of what the lake
contains; a register that reads as a summary of the research has failed even when
every line in it is true.

**Clustering the lake.** Group by subject, never by folder or by date. A
contradiction only surfaces when both halves reach the same prospector, so a
brief in `docs/` and a transcript in `meetings/` have to land together. Err
toward fewer, larger clusters: overlap is worse than size, because the same
source in two clusters returns the same claim twice and you cannot tell
corroboration from duplication. Failure looks like two entries citing one
transcript for opposite conclusions, neither mentioning the other.

**A prototype is a decision log.** Ask what the code decided, not how it works.
An accurate map of a prototype reads as a specification, which is exactly wrong,
because nobody reviewed those choices. Every platform choice becomes an
accept-or-override row, never a driver. Size picks the agent: `cartographer` for
an application spread across directories, `codebase-pattern-finder` for one route
handler somebody pasted in. Borderline goes to the cartographer, since
under-reading inherits decisions silently and over-reading only costs a context.

**Prior context.** Pull the archivist unless you are certain there is no trail.
It runs alongside the prospectors and costs no wall clock, and what it finds
changes the job: a prior register makes this run a diff, and a superseded plan
means re-proposing the reversed option will read as not having done the reading.
It returns a briefing, not a verdict, so cite the artifact rather than asserting
on its word.

**Hunting the quiet contradictions.** `lake-provenance.md` carries the six kinds.
The one thing it cannot give you is the search direction: sweep from each
governing commitment outward to the choices under it, never from the claims.
Reading "all buttons should be blue" tells you nothing because nothing about it
looks wrong; the standard is what tells you where to look. Most of what you find
resolves to a question with an owner, and picking a side is the worst available
outcome, because a settled-looking entry stops anyone checking.

**Which questions are actually for the human.** Ask only what another pass over
the lake cannot settle: which of two sources is authoritative, whether a document
was signed off, who owns a gap, whether a stated constraint still holds. The
error is asymmetric. Over-asking costs thirty seconds of irritation; under-asking
puts a guess into a document that looks sourced. Ask when unsure.

**A correction needs a locator too.** A correction and a locator that disagree
are two claims, and the locator is not automatically the wrong one. Usually the
source is stale, so find what makes it stale and cite that. Sometimes the user
is recalling a conversation that never entered the lake, and then they are the
locator, recorded as such rather than laundered into a document citation.
Occasionally they are wrong and the source says what you said it said: quote it
and let them decide. Where a correction invalidates a cluster's whole reading,
`retry:` back to `prospect` rather than patching entries by hand.

**Source authority is applied, not decided.** The order lives in
`lake-provenance.md`. Apply it and record the application; do not weigh conflicts
on their merits. Per-case judgment feels more careful and is less defensible,
because it drifts toward whichever source supports the cleaner architecture and
no reviewer can see that happening. Record both sides, both locators, the winner
and the rule, and never drop the loser: the overruled claim is often the one a
stakeholder remembers hearing. Where the order does not resolve a conflict, that
is an open question, not a coin flip.

**The line between derived and invented.** Assert what the research supports,
never what it implies to you. A lake yields domain nouns, actors, workflows,
stated numbers, non-goals and named integrations. It does not yield language,
framework, cloud service, database engine, service boundaries or transport; when
one appears it is somebody's aside and enters as an implicit decision to review.
Treat any resolution naming a technology as a defect until you can point at the
locator where a person chose it.

**The arbiter, not a relay.** The assayer's report is input. Accept anything
where it opened a locator and found something else there, which outranks your
memory of writing the entry. Push back where it flagged labelled uncertainty as a
defect: an assumption carrying an invalidation trigger is a correct entry, and an
assayer wanting everything decided has misread the artifact. Check a
`verdict: reject` yourself before acting, since it sends the pass back to the
lake and is the finding likeliest to over-react to a few bad entries. Relaying
findings unarbitrated is the failure; `owns: [arbitration]` says so.

**What the documents may contain.** The `documents:` map on `derive-docs` is
exhaustive, and `never-writes:` is why. `domain-model.md` carries mermaid
diagrams, which GitHub renders inline, but they are views of the cited tables and
never a source: draw only relationships whose cardinality a source states and
only transitions a source gives, because a drawn line asserts a cardinality and
mermaid has no notation for "unknown". Anything uncited stays in the table and is
listed as omitted, which is the finding. Nothing outside that list is created
unless a register entry names it with a locator, because a directory layout is a
technical decision wearing a filesystem costume. `technical-design.md` stays
structurally complete and substantively open; that is the deliverable, not a
shortfall. The asymmetry is the whole reason: a wrong extraction is visible to a
reader, a wrong dependency set is not, so the fix is to not generate it rather
than to label it conventional.
