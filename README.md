# groundwork

One hand-authored skill and two agents for **[hyprlayer](https://www.hyprlayer.dev/)**,
targeting Claude Code.

hyprlayer ([hyprlayer.dev](https://www.hyprlayer.dev/) ·
[BrightBlock/hyprlayer-cli](https://github.com/BrightBlock/hyprlayer-cli)) is a
CLI that installs a set of skills and agents into your coding agent, gives them a
shared notes store across git, Obsidian, Notion and Anytype, and validates the
declarative `orchestration:` blocks those skills use to wire up their sub-agents.
This repo adds one skill to that set.

The files here do nothing until they are copied into `~/.claude/`, and nothing
here ships with hyprlayer itself. These are additions that sit alongside it.

**[Skip to Install](#install)** if you just want to run it.

- [What it does](#what-it-does)
- [What you get](#what-you-get)
- [Install](#install) and [Layout](#layout)
- [How it works](#how-it-works)
- [The declarative block](#the-declarative-block), [Validate](#validate), [Requirements](#requirements)
- [Design decisions](#design-decisions-worth-knowing), [Not implemented](#not-implemented)

---

## What it does

`groundwork` reads a pile of product and design research (meeting
transcripts, briefs, PDFs, slide decks, spreadsheets, whiteboard photos,
vibe-coded prototypes) and produces a **decision register**: the architecture
decisions the research supports, each one citing the file and page it came from,
each gap named and assigned to somebody, and every argument it cannot settle
written down as a question rather than guessed at.

Optionally it then renders that register into a repo's documentation.

```bash
/groundwork --lake ~/research/atlas --out ~/code/atlas   # register + docs
/groundwork --lake ~/research/atlas                        # register only
```

The folder of research is called the **lake** throughout. `--lake` is required.
`--out` is optional and controls the entire document-writing half; without it you
get the register and nothing else, which is a normal way to run it. It only ever
runs when you type it, never on Claude's own initiative.

## What you get

**A register**, saved in your hyprlayer thoughts store (the notes store hyprlayer
manages, whichever backend you have configured): which files were read and which
could not be, the domain itself (glossary, what relates to what, state machines,
who can do what), hard constraints with what would invalidate each, the
decisions, open questions with owners, prototype choices to accept or override,
referenced standards, unsettled arguments, and settled arguments including the
losing side.

**A read-log**, so a second run does not start from scratch.

**With `--out`, six files in the repo:**

| File | What is in it |
|---|---|
| `docs/domain-model.md` | Glossary, what relates to what, state machines, who can do what |
| `docs/constraints.md` | The checkable statements: numbers, non-goals, integrations, and what would invalidate each |
| `docs/technical-design.md` | Mostly empty on purpose |
| `docs/open-questions.md` | The owner table |
| `docs/decisions.md` | A flat "we chose X because [source]" list |
| `CLAUDE.md` | Thin: the vocabulary and the non-goals |

`technical-design.md` being mostly empty is the point. The Transport section does
not say "Azure Service Bus." It says: pull-based per D3, fifteen-minute window
per C7, at-least-once versus exactly-once still open and Dana's call. Homework,
already researched.

**It never writes** a package list, a lockfile, CI config, a Dockerfile, a
language choice, or a folder skeleton, unless a decision in the register names it
and cites a source.

The reason is that the two mistakes are not equally visible. A wrong fact gets
caught: a reader sees "requests are pull-based" and says no, we decided the
opposite. A wrong package list does not, because a `.csproj` with twelve packages
looks like every other one, and nobody audits a dependency list against a pile of
meeting notes. So the fix is to not write it, rather than to write it and label it
a guess.

## Install

```bash
mkdir -p ~/.claude/skills/_thoughts/templates ~/.claude/agents
cp -R claude/skills/groundwork                  ~/.claude/skills/
cp    claude/skills/_thoughts/lake-provenance.md      ~/.claude/skills/_thoughts/
cp    claude/skills/_thoughts/templates/decision-register.md ~/.claude/skills/_thoughts/templates/
cp    claude/agents/prospector.md claude/agents/assayer.md   ~/.claude/agents/
```

The `_thoughts/` files must go to that exact path. `loads:` in a skill block
resolves against `~/.claude/skills/_thoughts/`, hardcoded in
`orchestration-runtime.md`, so there is no alternative location for a protocol
file or a template.

### Your files survive hyprlayer installs

hyprlayer ships its own skills, agents and hooks as a versioned **bundle** that
`hyprlayer ai configure` unpacks into `~/.claude/`. Two of the directories we add
to, `~/.claude/skills/_thoughts/` and `~/.claude/agents/`, are directories that
bundle manages, so it is worth knowing why adding to them is safe.

Orphan removal only iterates files listed in `~/.claude/.hyprlayer-manifest.json`,
and only deletes one whose bytes still match the digest recorded there. A
hand-authored file is never in the manifest, so it is never a deletion candidate.
Separately, the install's file sync preserves any file whose bytes match neither
the incoming bundle version nor the last recorded digest, reporting it as
`Kept your modified <path>`.

The practical rule: **a name the bundle does not ship is safe.** All five names
here qualify. Do not add a file at a path the bundle already owns unless you
intend to freeze your copy of it, because a local edit survives but then silently
stops receiving upstream updates.

## Layout

```
claude/
  skills/
    groundwork/SKILL.md                    the pipeline: 24 steps, 2 approval gates
    _thoughts/
      lake-provenance.md                   citations, extraction, contradictions, authority
      templates/decision-register.md       the register's body structure
  agents/
    prospector.md                          reads one group of documents, returns sourced claims
    assayer.md                             tries to break a drafted register
```

The layout mirrors hyprlayer's own `assets/claude/` tree minus the `assets/`
prefix, so it copies straight across if any of it is contributed upstream.

---

## How it works

### What happens when you run it

You point it at a folder of research. It works through nine stages, showing you
its working twice and asking for approval twice. A full run is 21 waves of work.

1. **Look at everything in the folder.** Count the files, work out how each one
   can be read, and say up front how many cannot be read at all.
2. **Group the files by topic.** Not by folder.
3. **Read every group at once.** One reader per group, all in parallel. Each
   comes back with claims, and every claim says which file and which page it came
   from.
4. **Go read the sources yourself.** The readers cite; the skill checks.
5. **Hunt for disagreements**, then show you what it found. *(shows you)*
6. **Fill the gaps.** A second reading pass aimed only at what round one left
   open.
7. **Settle what can be settled**, and show you the proposed decisions.
   *(shows you)*
8. **Write the register, then have a second agent attack it.** *(asks approval)*
9. **Save it**, and if you asked for it, write documents into a repo.
   *(asks approval)*

### It reads anything

There is no supported-formats list. It tries these in order and stops at the
first one that works: just read the file; if the file is secretly a zip, unzip it
(Word, PowerPoint, Excel and Visio files all are); if macOS can convert it,
convert it; if it is a data file, read the rows; if it is something odd, ask
`file` what it is and work from there; as a last resort, scrape the printable
text out and mark anything found that way as unreliable.

An unfamiliar file extension is never a reason to skip a file. If it genuinely
cannot be read, that gets recorded rather than quietly dropped. The full
procedure is in [Requirements](#requirements).

**One thing it cannot do is audio and video.** Nothing on a stock Mac transcribes
them. If your folder is meeting recordings rather than transcripts, it tells you
that first thing, because then the main source is missing entirely and everything
else is beside the point.

### It groups by topic, not by folder

You only notice a disagreement if both halves land in front of the same reader.
If the brief lives in `docs/` and the transcript lives in `meetings/`, grouping by
folder means the two sides of an argument never meet, and both end up in the
register as confident facts.

### Every claim says where it came from

Instead of "requests are pull-based," you get "requests are pull-based, per
`2026-03-04-standup.vtt @14:32 Dana`." So the PM who wrote the brief can go and
listen to minute 14 and tell you Dana was thinking out loud and nobody agreed.

The rule: **if it cannot say where a claim came from, it is not a claim, it is a
question.** An unsourced line is worse than a missing one, because on the page it
looks exactly like a real one and gets believed.

### It hunts for disagreements, including the quiet ones

The obvious kind is two documents giving different numbers. Those are rare, and
the team usually knows about them already. The useful kind is quieter.

Say someone in a meeting says *"all buttons should be blue."* A different
document says *"this follows the Acme design system."* Neither sentence
mentions the other. Neither is wrong on its own. But the design system governs
button colour, and nobody checked whether it says blue.

That kind matters because **the specific choice is the one that gets built.**
Somebody writes the blue button on Monday. The design system reference sits in a
document nobody reopens. The argument happens at design review six weeks later,
when it costs a sprint instead of a sentence.

To catch it, the skill collects every promise to follow some outside standard (a
design system, a coding standard, an architecture rule, a compliance regime, an
old decision record) and then looks for choices that fall under each one. **That
direction is the whole trick**, because the blue button never looks wrong on its
own. The standard is what tells you where to look.

Six kinds get hunted:

| Kind | Example or shape |
|---|---|
| Straight disagreement | "Fifteen minutes" against "an hour" |
| **A choice against a standard** | "Buttons should be blue" against "follows the Acme design system" |
| Split across groups | "Fifteen minutes" and "an hour" said in two different meetings, landing in two different topic groups |
| Scope | "Not doing offline in v1" against a deck full of offline screens |
| Two things that cannot both be true | "Real-time dashboard" against "nightly batch load" |
| Against what the prototype did | A brief says single-tenant, the prototype has a `tenant_id` column |

#### What it does with them: nothing, on purpose

It writes a row under **"Needs a resolution on"**: both sides with their sources,
what it is blocking, and whose call it is. Blue cannot be settled against a design
system that is not in the folder, and picking either side would produce a
settled-looking answer that stops anyone from checking.

It also lists every outside standard that gets referenced but is not in the
folder, even where it found no argument. That is where the arguments it missed are
hiding, and it tells you which document to go and find.

### When two sources really do disagree

The order is fixed in advance:

- A signed-off document beats a draft of the same document.
- Same kind of document, later beats earlier.
- **A transcript shows people talking, not deciding**, unless somebody in it says
  "we decided X."
- A prototype shows intent that nobody reviewed.
- A number written down beats a number said out loud.

Fixed in advance so it can be checked. Judging each argument on its merits sounds
more careful but leans toward whichever side gives the tidier architecture, and
nobody reading the register can see that happening.

The losing side is always written down. It is usually the one somebody remembers
hearing, and a register that never mentions it reads like it missed the meeting.

### Prototypes are not evidence

A vibe-coded prototype has already picked login, database, state handling and
hosting, and nobody reviewed any of it. Read as input, the register turns whatever
it happened to use into policy.

So what gets asked of a prototype is "what did this decide?" rather than "how does
this work?", and every answer becomes an accept-or-override row. "The prototype
kept state in the component" is not a decision. It is a question with a default
already filled in.

### It stops for you four times

Twice it shows you its working, and twice it asks for a yes before continuing.

- **After the claims, before any decisions** *(shows you)*. The cheapest moment
  to say that Dana left and her brief is stale, or that the deck it leaned on was
  never approved. If your correction breaks a whole group's reading, it goes back
  and re-reads rather than patching around it.
- **After the decisions, before anything is written** *(shows you)*. You assign
  the owners here, because it cannot. Most gaps belong to the PM who wrote the
  brief or the researcher who ran the study.
- **Before saving the register** *(asks approval)*.
- **Before it touches your repo** *(asks approval)*. Approving the register is
  not approving the files.

Only the last two are hard gates in the block; the first two are review points
you can talk back to.

### It gets attacked before you read it

`assayer`, a second agent, tries to break the register before you see it, because
a register you have already read is one you have already started trusting.

It opens each citation and checks the source actually says what the register
claims, since a citation pointing at the wrong page is worse than none: it
survives review by looking precise. It also checks whether "decided" entries are
really guesses, whether anything was decided against a standard nobody read,
whether two entries contradict each other, whether guesses admit to being
guesses, and whether every open question has a name against it.

Its verdict is ship, revise, or reject. The skill treats that as an opinion
rather than a ruling: an entry that says "we are assuming X, and here is what
would prove it wrong" is a *correct* entry, and a reviewer demanding everything be
settled has misread the job.

### Running it again

Answering an open question usually unblocks a decision, and a changed decision
goes back to the `assayer`. There is no cap on rounds. A register is something you
circulate and revise, not a one-shot output.

New files re-enter at the top, and the read-log is what keeps that cheap: it
re-reads what changed rather than all three hundred documents.

### The two agents

Neither can spawn another agent or call a skill. That is hyprlayer's rule, and it
is why the grouping, the judging and all the saving stay with the skill itself. A
reader reads its group and reports back; it cannot go and recruit another reader.

**`prospector`** reads one group of documents and reports claims, referenced
standards, disagreements, decisions nobody announced, and gaps. Hand it a folder
instead of a file list and it stops rather than guessing, because grouping is the
skill's job and a group it invented would overlap somebody else's and count the
same claim twice.

**`assayer`** takes the finished register and tries to break it. It does not fix
anything or suggest wording, because the skill needs its findings kept separate
from the skill's own opinion.

It also borrows three agents hyprlayer already ships: `cartographer` or
`codebase-pattern-finder` to read a prototype, whichever suits its size, and
`archivist` to dig up any earlier register on the same topic.

---

## The declarative block

*This section is for anyone editing the pipeline. Skip it to run the skill.*

From here on, the files' own vocabulary: a **group** is a `cluster`, a **reader**
is a `prospector`, and a **citation** is a `locator`.

The whole pipeline is one `orchestration:` block, and `hyprlayer orchestrate`
validates and schedules it. It never executes it: running the steps is the
model's job.

**`steps:` is a set, not a sequence.** Nothing in the block is numbered. Each
step names what it needs via `requires:`, and the scheduler derives the order and
finds the parallelism. That is why wave 4 runs five prospectors, a cartographer
and an archivist at once, and it is also why a mistake in `requires:` produces a
plan that is wrong rather than one that fails.

What the block carries:

- **Eight `when:` guards.** No `--out` and the entire document half skips. No
  prototype and `map-prototype` skips. Notion instead of git and `sync` skips.
  Nothing left open after round one and `deeper-prospect` skips.
- **Two `fanout:` steps** over lists derived at runtime, one per cluster.
- **Six `reject:` preconditions.** `write-docs` re-checks that you actually
  approved rather than assuming so because the approval step ran, since a
  satisfied `requires:` proves a step ran, not that its check passed.
- **Two `retry:` rules**, letting a user correction re-trigger research.
- **One unbounded `re-challenges:`** on `iterate`.
- **Eleven `unresolved[]` entries**: ten judgment calls plus one agent choice,
  recorded as decisions the block declines to make so the compiled plan reads as
  a to-do rather than pretending it decided.

One authoring gotcha worth knowing: the parser validates exactly twelve fields
(`id`, `requires`, `agent`, `fanout`, `over`, `when`, `when-examples`,
`judgment`, `reject`, `given`, `retry`, `inline`). Everything else in the block,
including `owns:`, `asks-user:`, `presents:`, `loads:`, `artifact:` and
`because:`, is convention the model honours and the validator ignores completely.
A typo in `asks-user:` passes `check` silently and shows up as a gate that never
fires.

### What it caught

Three real bugs surfaced from reading compiled plans, none of which were visible
in the file:

1. **No parallelism at all.** The first draft was 23 steps in 23 waves. That is
   what prompted adding `prior-context`, which does useful work in an existing
   wave.
2. **Files written before the register existed.** On a register-only run
   `derive-docs` skips, and a skipped step *satisfies* its dependents, so the
   entire document tail unblocked immediately and landed in waves 1 to 3. Every
   step in that tail now carries its own `when: flag(--out)`, and `show-result`
   requires `save` as well as `write-docs` to anchor it.
3. **The same bug one step down.** Adding the `count(follow-up-clusters) > 0`
   guard let `deeper-prospect` skip, which floated `reconcile` to wave 1.
   `reconcile` now requires `verify-corrections` too.

## Validate

```bash
hyprlayer orchestrate check claude/skills/groundwork/SKILL.md \
  --target claude --agents-dir claude/agents --agents-dir ~/.claude/agents
```

`check` runs six numbered checks over the block. Number 6 is agent-name
resolution, and number 4 evaluates each guard against its own worked examples.

**`--agents-dir` replaces the agent namespace rather than adding to it.** Pass it
only `claude/agents` and check 6 fails with three errors: `unknown agent
cartographer`, `unknown agent codebase-pattern-finder` and `unknown agent
archivist`, because those are hyprlayer's and live in `~/.claude/agents`. The flag
is repeatable, so pass both. Once the two agents here are installed, drop the flag
and the default namespace resolves all five.

hyprlayer's own worked example gets away with a single `--agents-dir` because
`assets/claude/agents` holds every agent. A repo carrying only new ones does not.

Expect `ok` with eight warnings, all `[check 4] examples cannot be evaluated
statically`. That is the normal result for guards using `flag()`, `exists()`,
`count()` and `backend`, which resolve at compile time rather than check time.
`create_plan`, one of hyprlayer's own shipped skills, produces the same class of
warning.

Then read the schedule:

```bash
hyprlayer orchestrate compile claude/skills/groundwork/SKILL.md \
  --target claude --agents-dir claude/agents --agents-dir ~/.claude/agents \
  --fanout clusters=5 --fanout follow-up-clusters=2 \
  --fact 'flag(--lake)=true' --fact 'flag(--out)=true' \
  --fact 'exists(lake.prototype)=true' --fact 'exists(manifest)=true' \
  --fact 'backend=git' --fact 'count(follow-up-clusters)=2' --human
```

Both fanout counts are required, because the block has two. `count()` facts pin
as **integers**, not booleans: `count(follow-up-clusters)=2`, never `=true`. Pin
the facts rather than probing if you want to compare runs, since the same block,
facts and fanout counts produce a byte-identical plan and a stable `planHash`.

| Run | Steps | Waves | Spawns | Skipped |
|---|---|---|---|---|
| Full (`--out`, prototype, warm manifest, git) | 24 | 21 | 10 | 0 |
| Register only (no `--out`, cold, notion) | 24 | 15 | 7 | 8 |

As `orchestration-runtime.md` puts it, neither command proves the block is right.
Read the wave listing and confirm the parallelism and skip reasons are what you
meant, and read `unresolved[]` and confirm every declined decision is one you want
left to the model.

### The check runs itself

`.claude/settings.json` carries a `PostToolUse` hook on `Write|Edit` that runs
`orchestrate check` on any file whose path ends in `SKILL.md`. A clean block is
silent; a broken one exits 2 and feeds the findings straight back, so a bad
`requires:` or an unresolvable agent name surfaces at the edit rather than at the
next manual run.

It derives `--agents-dir` from the edited file's own path rather than hardcoding
one, so it works in any checkout: `${f%/skills/*}/agents` resolves
`<root>/claude/skills/<name>/SKILL.md` to `<root>/claude/agents`. Being
side-effect-free is what makes `check` safe to run this way, and the upstream
commit says so explicitly.

The hook only loads for sessions whose project root is this repo, and Claude Code
only watches directories that already had a settings file when the session
started. After pulling this for the first time, open `/hooks` once or restart.

## Requirements

No document converters need installing, and **no file type is out of scope**.
`lake-provenance.md` carries an ordered extraction procedure rather than a list of
supported formats, using only what macOS already has:

| Step | Handles | How |
|---|---|---|
| 1 | Text, markdown, code, PDF, images | `Read`. A PDF page range doubles as the citation; reading an image beats OCR, which a photographed whiteboard defeats |
| 2 | `.pptx` `.docx` `.xlsx` `.vsdx` `.odt` `.epub` `.zip` | They are all zips of XML. `unzip -l` to see, `unzip -p` to read. Slide and sheet numbers come free from the internal filenames, and `.docx` heading paths come from the `w:pStyle` values |
| 3 | `.doc` `.rtf` `.rtfd` `.webarchive` | `textutil -convert html`, for the non-zip legacy formats only. It flattens headings to `<p>`, so the citation drops to a paragraph ordinal |
| 4 | `.csv` `.tsv` `.json` `.xml` `.yaml` `.log` `.sql` | Read as data; the citation is the row or key path |
| 5 | Anything unrecognised | `file` to identify it, then read it accordingly. `strings` as last resort, and any claim from it is flagged low confidence |
| 6 | Audio and video | Cannot be read. Marked unread and surfaced loudly, because a lake of recordings with no transcripts means the primary source is missing |
| 7 | Everything else | Unread, recorded in the read-log with the step it failed at |

An unfamiliar extension is never a reason to skip a source. Steps 5 and 7 exist
so an unknown format gets attempted and then honestly reported rather than
quietly assumed unreadable. pandoc, markitdown, tesseract and libreoffice are
deliberately not used.

Step 2 and step 3 were both verified against real files. The `.docx` heading path
reconstructs correctly from `w:pStyle`; `textutil` was found to flatten every
paragraph to `<p class="pN">`, which is why Word routes through step 2 instead.

---

## Design decisions worth knowing

**The register is `type: plan`.** A register of proposed decisions is a plan,
which is honest and needs no change to `THOUGHT_SCHEMA`, the one list of document
types that hyprlayer (a Rust CLI) shares across every user and every storage
backend. Adding a `type: adr` option would mean migrating that list, and existing
Notion and Anytype users have live dropdown properties built from the old values.
That belongs to the maintainer, not a skill author.

The cost of borrowing the label: `implement_plan` and `validate_plan`, two of
hyprlayer's own skills, will happily pick the register up and then find no phases
and no test commands. Nothing in the Rust validates a document's body against its
declared type.

**Not ADRs, at least not first.** An ADR records a decision a person made at a
time and accepted the consequences of. This is a decision *inferred from
documents*, and writing inference in ADR format launders it into authority. Forty
ADRs dated the same day with the same author is a smell experienced people
correctly distrust, and the format's most valuable field, Consequences, is
precisely the one research cannot fill. After you review the register and accept a
subset, *those* become real ADRs, because now a human decided. That is a good
second command, not the primary output.

**The register saves with its open questions intact**, unlike hyprlayer's
`create_plan`, which refuses to save a plan that still has one. Here the owned
questions are the deliverable.

**One skill, not two, and not a session agent.** `create_plan` settles the
argument: it has three sequential human gates, uses `retry:` as a genuine
loop-back so a correction re-triggers research, and its `iterate` step re-runs
its reviewer with no cap on rounds. Ask, answer, re-derive, ask again, unbounded,
in one block, in a shipped skill. `create_plan`'s `owns:` list even names
`user-dialogue` outright.

The block also has no primitive for calling another skill. The step vocabulary is
`inline`, one `agent`, or a `fanout` of one agent type. Skills *can* invoke skills
through the `Skill` tool, but the block cannot express it, so `orchestrate` would
schedule and validate a plan that lies about what runs.

## Not implemented

**Tiered decisions across team and company.** Decisions shared at a team level
and fewer still at a company level, where an overriding constraint renders
visibly and names what it departs from. Cut because hyprlayer's `scope` field
(`user | shared | global`) is not an org hierarchy, so there is no storage model
for the tiering, and because the schema can express *replaced* via `related` plus
`status: superseded` but not *narrows* or *grants an exception to*, which is the
entire value of tiered decisions. Revisiting it needs both: a storage model that
is not `scope`, and a typed relation expressing narrowing.
