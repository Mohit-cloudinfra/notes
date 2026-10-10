# The Bible — how to work in `D:/notes/`

This folder lives at `D:/notes/`, outside the `D:/inprogress` repo (moved 10 Oct 2026). Who
Sai is, his goal and how he wants to be worked with are in `D:/inprogress/ABOUT-ME.md`.

Sai is building one reference he revises from, which he calls his **bible**. It lives in
`D:/notes/my notes/`. When he says **"I uploaded new photos"** / **"new notes"**, run the
routine below. Read this whole file first.

## What the bible is

- **Organised by topic, never by session.** `my notes/<subject>/NN-topic.txt`. Sir's
  "session 1, session 2" order is ignored; session numbers appear only in `STATUS.txt`.
- **Plain `.txt`**, not Markdown. He reads it in VS Code and wants it to look good there.
- **It only covers what he has actually studied.** Sir's notes exist for all 84 sessions,
  but writing ahead of him turns it into a textbook instead of his notes. Evidence that he
  has studied something: handwritten photos, his own run logs, or him saying so.
- Built on 10 Oct 2026, covering sessions 1–12. `my notes/STATUS.txt` says where it is now.

## The sources and what each is for

| Path | Role |
|---|---|
| `DevSecOps with AWS - 88S.txt` | Sir's syllabus. **A checklist**, used to spot topics nobody wrote down. |
| `sir notes/session-NN.txt` | Sir's class notes. Patchy: sometimes detailed, sometimes three lines, sometimes absent (no session 80). A gap does not mean nothing was taught. |
| `handwritten/` | Photos of Sai's own handwritten notes. New batches land here or as a zip in `D:/notes/`. |
| `sir documents/` | Sir's official step-by-step guides for Expense and Roboshop. The correct procedure. |
| `my documents/` | Sai's own terminal logs from doing the practicals. Mine these for the errors he really hit. He adds to this folder. |
| `my notes/` | **The bible.** Nothing else lives here. |

## Routine: "I uploaded new photos"

1. **Find the new batch.** A zip in `D:/notes/`, or loose images in `D:/notes/handwritten/`.
   Anything already under `handwritten/processed/` is done. Extract a zip into
   `handwritten/` first. Also check `my documents/` and `sir documents/` for new files.
2. **Read every photo.** Then read the matching `sir notes/` sessions and the syllabus
   entries for those sessions.
3. **Report to him before writing, in this order, short:**
   - **Doubts** he wrote in his notes, each with an explanation. *Explain first. A doubt
     goes into the bible only after he says he has understood it.*
   - **Mistakes** in his notes, with the correction.
   - **Proposed additions** — things neither he nor sir wrote that would help. *Ask. Add
     only what he approves.*
   - **Syllabus topics with no notes at all** for those sessions.
4. **Merge into the right topic files.** Add to an existing file where the topic exists;
   create a new numbered file or a new subject folder where it does not. Never append a
   "batch 2" chunk at the end of a file.
5. **Update `my notes/STATUS.txt` and `my notes/SUMMARY.txt`.**
6. **Move the processed photos** (and the zip) to
   `handwritten/processed/YYYY-MM-DD/`, dated by the batch.
7. **Tell him what changed**, in a few lines: files touched, what is still open.

## Content rules

- **Only add. Never remove.** He said this directly on 10 Oct 2026. Nothing already in the
  bible gets deleted, trimmed or condensed when new material is merged in. New content goes
  alongside what is there. If something existing is wrong, fix that line and tell him. If
  something looks redundant or should be reorganised, ask him first. The same goes for
  files and folders in `my notes/`: do not delete or rename them without his say.

- **How to decide what goes in.** Sort every line you are about to write into one of four
  kinds:
  1. **From class** — it is in his handwritten notes, sir's notes, sir's documents or his
     practical logs. Goes in, unmarked. No judgement call: if it is in a source for a
     session he has studied, it is in the bible.
  2. **A correction** — a source says it, but it is wrong. Goes in corrected, unmarked,
     and he is told.
  3. **An explanation of something already there** — one line saying what a flag, a field
     or a step does (`-y = say yes`, what `ExecStart` is), the `IN ONE LINE` and `REMEMBER`
     blocks, and the diagnosis of an error from his own logs. Goes in unmarked, kept to a
     line or two. It adds no new topic, it only makes an existing line readable.
  4. **A new fact** — a command, concept, port, behaviour or interview point that no
     source mentions. **Never goes in on your own.** Propose it, explain it if he does not
     follow, and add it marked `[+]` only after he says yes.

  The test between 3 and 4: *if this line were deleted, would a topic disappear, or would
  an existing line just be harder to read?* If a topic disappears, it is kind 4. When
  unsure, treat it as kind 4 and ask.

  What makes a kind 4 worth proposing: it fixes a gap that would stop him doing the task
  (a missing step or setting), it is something he will hit as an error, or it is commonly
  asked in interviews for his target level. Do not propose things that are merely
  interesting.
- **Miss nothing.** Every line in his handwritten notes and in sir's notes for the covered
  sessions goes in somewhere. A fragment with no recoverable meaning is listed back to him
  as left out; it is not silently dropped.
- **Keep his wording and sir's stories.** The restaurant, the DM and team lead, the exam
  story, the production incident. These are what he recalls in interviews. Do not flatten
  them into textbook language.
- **Corrections go in corrected.** If his notes or sir's notes are wrong, the bible carries
  the right version, and he is told what was changed.
- **Additions need his yes.** Approved additions are marked `[+]` at the start of the line.
  Everything unmarked came from class.
- **Answered doubts** sit where they belong under a `MY DOUBT, ANSWERED` heading, as Q and A.
- **His real mistakes** from `my documents/` go under a "mistakes I actually made" heading,
  with the exact error text and the fix.
- **No real credentials.** Sir's notes contain Gmail app passwords, an AWS account ID and
  similar. Leave them out. Lab defaults that are part of a documented command
  (`ExpenseApp@1`, `RoboShop@1`, `roboshop123`, the practice AMI login) are fine.
- Cross-reference instead of repeating: `More: networking/02-dns.txt`.

## Format of a topic file

Match the existing files exactly. Open `my notes/linux/06-permissions-and-ownership.txt`
as the reference.

- Width 72 characters. Header and footer are a line of `=`. Section rules are a line of `-`.
- Title line: `  SUBJECT  >  TOPIC` in capitals.
- First block is `IN ONE LINE`: the whole topic in one or two sentences.
- Numbered sections, each with a capitalised heading.
- Commands indented, with a short plain explanation to the right or below.
- ASCII diagrams where his notes had a drawing.
- Second-to-last block is `[+] INTERVIEW` (see below).
- Last block is `REMEMBER`: three to five lines he should be able to say from memory.

## The INTERVIEW section

His goal is to clear interviews, not to collect knowledge. Sir's notes get him able to
build the thing; they are thin on the why, on what breaks, and on the follow-up question.
The `INTERVIEW` section fills that, and only that. He approved the format on 10 Oct 2026,
so **add it to every topic file without asking each time**, and to every new topic file
you create.

- **Place:** just before `REMEMBER`. Heading `[+] INTERVIEW`, each question `[+] Q1.` and
  so on. Match `my notes/linux/09-services-ports-and-systemd.txt`.
- **Size:** 3 to 6 questions per file. Each answer short enough to say in under a minute.
  Do not go wider; volume is what makes him stop reading.
- **Which questions:** the ones actually asked on that topic at **SRE-1 / junior DevOps**
  level. Favour "what is the difference between X and Y", "X is failing, what do you
  check", and "why did you do it this way". Not trivia, not deep internals.
- **`FROM MY OWN RUN`:** tie an answer to something he really did, using his logs in
  `my documents/` or his notes. **Only claim what the log shows.** If his log shows the
  error but not the fix working, say so in a `NOTE TO SELF` line instead of writing "and
  then it worked". An answer he cannot back up falls apart on the second follow-up.
- **`SAY THIS`:** a ready sentence for "explain X" questions.
- **`BE HONEST`:** wherever the true answer is "this was a lab". He must say lab himself.
  Never help him imply production experience (see `D:/inprogress/ABOUT-ME.md`).
- **One-line alternatives are allowed here** when an interviewer or a real server will
  force them (`ss` for `netstat`). Not otherwise.
- **Still no topics from sessions he has not reached.**
- When a later batch adds to a topic, add questions to the existing section; do not
  remove old ones.

## `STATUS.txt` and `SUMMARY.txt`

- **`STATUS.txt`** — per subject, each syllabus topic marked `done`, `thin` or `not yet`,
  with the file it lives in, a progress bar, the photo batch log and open items.
- **`SUMMARY.txt`** — the whole bible on one page: every file with a three or four line
  gist. Add an entry whenever a file is added.

## Quizzing

"I want to remember" (see `D:/inprogress/CLAUDE.md`) now reads from the bible in `my notes/`,
not from `sir notes/`.
