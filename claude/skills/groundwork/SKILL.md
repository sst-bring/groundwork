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
        See "Hunting the quiet contradictions" below.
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
      documents:                    # exhaustive: nothing outside this list is written
        "docs/domain-model.md":     "glossary, relationships, state machines, actors, each with a locator"
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

## Judgment

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
exhaustive, and `never-writes:` is why. Nothing outside that list is created
unless a register entry names it with a locator, because a directory layout is a
technical decision wearing a filesystem costume. `technical-design.md` stays
structurally complete and substantively open; that is the deliverable, not a
shortfall. The asymmetry is the whole reason: a wrong extraction is visible to a
reader, a wrong dependency set is not, so the fix is to not generate it rather
than to label it conventional.
