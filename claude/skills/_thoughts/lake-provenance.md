# Lake provenance

Binds you and every agent you spawn when the source material is a document lake
rather than a code tree. `documentarian-rules.md` assumes `file:line` is
available as ground truth. A lake has no equivalent, so the rules below replace
that half of it. Everything else in `documentarian-rules.md` still holds:
describe what the sources say, never what they should have said.

## Locators

Every claim carries a locator: **the smallest addressable unit of the source
that a reader of that format would use to find the claim again.**

That sentence is the rule. The table below is worked examples, not an allowlist.
No file type is out of scope by default. For a format not listed, apply the
rule: find the subdivision the format itself names, and cite it the way that
format's own tooling names it.

When a format offers no named subdivision, walk down this ladder and stop at the
first rung that holds:

1. **A named subdivision.** Page, slide, sheet and cell, heading path,
   timestamp, message id, commit and line.
2. **An ordinal position.** "The fourth table," "record 212," "the second
   attachment."
3. **A byte or line offset.** `notes.log:1841`.
4. **The whole file.**

A whole-file locator is legal and weak. When you use one, say so in the claim,
because a reader needs to know they are being asked to read an entire file to
check you rather than jump to a line.

Worked examples:

| Source | Locator | Example |
|---|---|---|
| PDF | page number | `pricing-study.pdf p.14` |
| Word doc, markdown | heading path | `platform-brief.docx > Scope > Out of scope` |
| Meeting transcript | timestamp plus speaker | `2026-03-04-standup.vtt @14:32 Dana` |
| Slide deck | slide number | `q2-review.pptx slide 7` |
| Prototype code | commit plus `path:line` | `a1b2c3d src/api/pull.ts:41` |
| Spreadsheet | sheet plus cell range | `volumes.xlsx > Forecast!B4:B9` |
| Image, screenshot, whiteboard photo | filename plus what is visible | `whiteboard-0312.jpg, upper left flow` |
| Email | sender, date, subject | `msg from Dana 2026-03-04 "re: batch window"` |
| Structured data | row, record or key path | `volumes.csv:212`, `config.json > limits.batch` |
| Diagram | the named shape or lane | `flow.vsdx > Ingestion lane, "retry" box` |
| Anything else | the best rung of the ladder above | `archive.zip > notes/day2.txt:14` |

## The hard rule

**An unlocatable driver does not enter the register as an assertion.** It
becomes an open question with an owner.

This is the rule the whole artifact rests on. Without it the register produces
confident architecture grounded in something someone speculated about in minute
34 of a call, which is worse than producing nothing, because a sourced-looking
claim is trusted and an absent claim is investigated.

If you believe a claim is true but cannot locate it, that belief is the open
question. Write the question, name who would know, and move on.

## Extraction

**No file type is out of scope, and no external converter is required.** This is
a procedure, not a list of supported formats. Work down it per source and stop
at the first step that yields text. Do not decide a file is unreadable because
its extension is unfamiliar; decide it at step 7 or not at all.

1. **Read it.** `Read` handles text, markdown, source code, PDFs (pass a page
   range, so the page-number locator comes free) and images (better than OCR,
   which a photographed whiteboard defeats and which you do not need, since you
   can see the image). Most of a lake stops here.

2. **If it is a zip, open the zip.** Every modern Office format and several
   others are zip archives of XML, and `unzip -l <file>` tells you in one
   command. This single step covers most of what a corporate lake actually
   contains:
   - `.pptx`: `unzip -p deck.pptx ppt/slides/slide7.xml`, and the slide number
     is guaranteed by the filename rather than inferred. List them with
     `unzip -l deck.pptx | grep slides/`.
   - `.docx`: `unzip -p doc.docx word/document.xml`. This is also where the
     heading path comes from: paragraphs carry `w:pStyle w:val="Heading1"`,
     `Heading2` and so on, in document order, so the path for any claim is the
     nearest preceding heading of each level. Do not use `textutil` for a
     `.docx` when you need a heading locator (see step 3).
   - `.xlsx`: `unzip -p book.xlsx xl/worksheets/sheet1.xml` together with
     `xl/sharedStrings.xml`, which holds the actual text
   - `.vsdx`, `.odt`, `.odp`, `.epub`, plain `.zip`: same move, different
     internal paths. `unzip -l` first, then read the parts that look like
     content.

3. **If it is not a zip but macOS knows it, convert it.** `textutil -convert
   html -stdout <file>` handles the legacy and non-zip formats: `.doc`, `.rtf`,
   `.rtfd`, `.webarchive`.

   **It flattens headings.** Verified on a real document: every paragraph comes
   out as `<p class="pN">`, so a Heading 1 is indistinguishable from body text
   and no heading path can be recovered. That is why step 2 handles `.docx`
   instead. When `textutil` is the only route, accept it and drop to the
   locator ladder: cite the ordinal paragraph or a line offset, and do not
   invent a heading path the output does not contain.

4. **If it is structured text, read it as data.** `.csv`, `.tsv`, `.json`,
   `.xml`, `.yaml`, `.log`, `.sql`. The locator is the row, record or key path.

5. **If you do not recognise it, ask the bytes.** `file <path>` names the
   format. Then work out how that format is read rather than concluding it
   cannot be: most things are a zip, a database, or text with a header.
   `strings <path>` is the last resort. It recovers usable text from many
   binaries, but a claim sourced from `strings` output is low confidence and the
   locator must say `via strings` so a reader knows the structure was lost.

6. **Audio and video cannot be read.** Nothing on a stock macOS box transcribes
   them, and guessing at the contents of a recording is the worst available
   outcome. Mark them unread, and surface the count **prominently** rather than
   in a footnote: a lake of twenty meeting recordings with no transcripts is not
   twenty unreadable files, it is the entire primary source missing, and the
   user needs to hear that before anything else. Ask whether transcripts exist
   somewhere else, which for Teams and Zoom recordings they usually do.

7. **Otherwise, unread.**

Record every unread source in the manifest, with which step it failed at and
what `file` said it was. An unread source is a known gap and costs little. A
hallucinated summary of one is a lie with a filename attached.

## Kinds of contradiction

Two sources saying opposite things is the easy case and the rarest. Most real
conflicts in a lake never mention each other. Hunt all six.

1. **Direct.** Incompatible assertions about the same fact. "The batch window is
   fifteen minutes" against "the batch window is an hour." Easy to spot, and
   usually already known to the team.

2. **Specific against governing.** A concrete choice that conflicts with a
   standard, system, policy or prior decision the material commits to somewhere
   else. *"All buttons should be blue"* against *"this follows the Simpson
   design system."* Neither sentence mentions the other and neither is wrong on
   its face. The conflict exists because the design system governs button colour
   and nobody checked whether it says blue.

   **This is the highest-value class and the easiest to miss**, because the
   specific choice is the one that gets built. Somebody writes the blue button,
   the commitment stays a sentence in a document nobody reopens, and the
   conflict surfaces at design review six weeks later.

3. **Cross-cluster.** A direct contradiction whose two halves sit in different
   clusters, so no single prospector ever saw both. Structural rather than
   subtle: catching it needs someone holding the whole set, which is the skill
   and never an agent.

4. **Scope.** Something one document lists as out of scope that another treats
   as a requirement. "Not doing offline in v1" against a deck full of offline
   screens.

5. **Consequential.** Two requirements that are each reasonable and jointly
   impossible. "Real-time dashboard" against "nightly batch load." Neither
   mentions the other; they conflict only in what they imply.

6. **Against an inherited decision.** A document says one thing and a prototype
   silently does another.

## Governing commitments

A **governing commitment** is any reference to a body of rules the lake does not
contain: a design system, a coding standard, an architecture tenet, a compliance
regime, a platform policy, a prior ADR or RFC. "Follows the Simpson design
system." "Per the platform tenets." "SOC 2 compliant." "Uses the standard auth
flow."

Treat every one as load-bearing, because it imports an entire set of constraints
by reference and any of them may contradict something stated specifically. Two
obligations follow:

- **List it.** Every governing commitment enters the register with its locator,
  whether or not it conflicts with anything yet.
- **List what it does not include.** A commitment whose content is not in the
  lake is a **referenced-but-absent source**, and the register says so
  explicitly. It is the single likeliest hiding place for an unfound
  contradiction, and naming it tells the reader precisely which document to go
  and fetch.

### What a specific-against-governing conflict resolves to

Usually nothing, and that is the correct answer.

You cannot resolve "buttons blue" against the Simpson design system without the
design system, and the design system is not in the lake. So it becomes an open
question with an owner, phrased as the decision somebody has to make rather than
as a topic: *"Do buttons follow the design system's colour tokens, or the blue
stated in the 4 March review? The design system is referenced but absent."*
Owner: whoever owns the design system.

**Do not pick.** Choosing blue because a person said it out loud is exactly what
the source-authority order forbids: a transcript is evidence of discussion, and a
stated commitment to a governing standard outranks it. Choosing the design
system is equally wrong, because you have not read it and it may well say blue.

## Source authority

Transcripts are the most abundant and least reliable input in any lake, because
people speculate out loud and reverse themselves inside a single meeting. Apply
this order rather than deciding case by case:

1. A signed-off design document beats a draft of the same document.
2. At equal type, later beats earlier.
3. **A transcript is evidence of discussion, not of decision**, unless someone
   in it states a decision as made. "We should probably pull rather than push"
   is discussion. "We decided pull, I will write it up" is a decision.
4. A prototype is evidence of intent that was never architecturally reviewed.
   See below.
5. A number written in a document beats a number said out loud.

When authority resolves a contradiction, **record what was overruled and why**.
Never silently drop the loser. The overruled claim is often the one a reader
remembers hearing, and a register that does not mention it reads as though it
missed the meeting.

## Prototypes are decisions to override, not inputs to trust

A vibe-coded prototype has already made twenty implicit platform decisions:
auth, data layer, state management, transport, deployment shape. Nobody
reviewed any of them. Treated as an input, the register canonizes whatever the
prototype happened to reach for on the day.

So every architecturally significant choice found in a prototype surfaces as an
explicit accept-or-override entry, with the override reasoned. "The prototype
used local component state" is not a decision. It is a question about state
management with a default already filled in, and the register's job is to make
that visible rather than inherit it.

## Blocking gap versus honest unknown

Two kinds of not-knowing, and the register must distinguish them.

An **honest unknown** is an entry that records an assumption plus the trigger
that would invalidate it. "Assuming under 500 concurrent users, because the
only figure anyone gave was aspirational. If the real number exceeds 5,000 the
transport decision reopens." That is a good entry. It is decidable, it is
labelled, and it names its own expiry.

A **blocking gap** cannot be decided without an answer nobody in the lake has.
The register says so and does not pick. An entry that silently guesses is the
failure mode this whole protocol exists to prevent, and it is indistinguishable
from a good entry by the time anyone reads it.

## Owner per open question

Most gaps are not answerable by the engineer running this. They belong to the
PM who wrote the brief or the researcher who ran the study. Every open question
carries a named owner, or "unowned" stated outright so the reader knows to
assign it.

The register is a document you circulate. It is not a prompt you answer.
