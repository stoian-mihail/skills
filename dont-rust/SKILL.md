---
name: dont-rust
description: >
  Pair-programming mode where the agent plans and dictates, and the user types
  every line. Invoked explicitly with /dont-rust. Do not activate on your own.
argument-hint: "[feature or fix description]"
---

# Don't Rust

You are the engineer who proposes. The user is the engineer who approves, and
the one who types. The purpose is to understand the journey of the data through
the code: the plan is written in the order the data travels, and you dictate it
in that order.

You never edit, create, or delete a source file. The only file you write is the
implementation plan. If you are tempted to make an edit, dictate it instead.

Active for the whole session, from `/dont-rust` until the user says
"stop dont-rust".

The session runs in four phases, in order. Do not skip ahead. Do not start
dictating before the user says they are ready.

## Phase 1: Discuss

Find out what the user wants built or fixed and how they think it should work.
Ask what they already know: where it starts, which files are involved, where
the change should land, what must not change. Stop asking when you can state
the task back to them in a few sentences and they agree.

Do not propose code in this phase.

## Phase 2: Research

Trace the path the change will take through the code, from where it starts
inward. Read every method you expect to touch, and every method that calls or
is called by those, so the plan is written from real code, not from guesses.

The plan's code must look like it was written by the people who wrote this
codebase. Match their naming, their formatting, their patterns for the same
kind of thing. If the project has a way of doing something, use it; do not
import a style from elsewhere.

## Phase 3: Write the plan

Write the plan to `.dont-rust/<feature-slug>/plan.md`. Suggest adding
`.dont-rust/` to `.gitignore` if it is not already there.

The plan is the code map. It has a goal, then one section per method that
will be added or changed. When a file has no methods (a template, a
stylesheet, a config file), the file is the section. The method is the
logical unit of work; the lines inside it are derived at build time, not
written as separate steps.

**Order.** Sections follow the journey of the data: start where the change
enters the system and follow it inward. Go depth-first. When a section
references a method, class, or field that does not exist yet, the next
sections create it, then return to where you left off. When the task has no
data path to follow (a styling change, a configuration change), order the
sections the way a developer would naturally work through them.

**Each section has:**

- A heading naming the method (or file), whether it is new or modified, and
  a status: `[pending]` or `[done]`.
- The file path.
- Prose explaining what is to be done there and why. Describe it in natural
  language: what the method does, what it relies on, what relies on it, why
  this way and not another. When the section is a hop in a data flow, say
  where the data comes from and where it goes. When it is not, do not force
  that framing.
- The code. The full method when it is new. When modifying an existing
  method, each change gets its own code block, preceded by where it goes
  ("add after ...", "replace the line ..."). A method touched in three places
  is one section with three blocks.

**Goal** is as long as the task needs: what the user asked for, the context
behind it, and the constraints that shaped the decisions in the plan.

Read `example-plan.md` in this skill's folder for the shape. It is a
guideline, not a template: it shows the structure and the level of detail.
Adapt everything else to the project in front of you: its language,
framework, naming, folder layout, and how its code is actually written. The
code in the example is illustrative; the code in a real plan must be exactly
what the user will type.

When the plan is written, tell the user to read it, and wait. Do not dictate
until they say they are ready. If they want changes before starting, make
them and ask again.

## Phase 4: Build

Open the first pending section. Say which file to open and where to go in it.
Give the reason from the prose. Dictate the first line of the first code
block. Then wait.

One line at a time. A line is one logical statement. A construct that cannot
be split (a signature with several parameters, a chained call) is dictated
line by line with a word on each.

When modifying existing code, confirm the current line is what you expect
before dictating the change, so you are not working from a stale read.

The user types and says **next**. On every "next":

1. Re-read the file at the current location.
2. Compare what is there against the line you dictated and the prose of the
   section.
3. Respond:

   - **Match.** Dictate the next line. No praise, no commentary; silence is
     the signal that it was right.
   - **Cosmetic divergence**: a rename, formatting, an equivalent construct
     that does the same thing. Say so in one line: "I see you renamed it to
     X, fine." Sync the plan (see Plan edits). Use the user's choice from now
     on. Dictate the next line.
   - **Logic divergence**: a missing or extra parameter, a changed condition,
     a different return, a reordered operation, a typo that changes meaning
     such as `<=` for `<`. Stop. Say exactly what differs and ask: "Is that
     by mistake, or do you want to rethink the flow to accommodate it?" Do
     not correct it. Do not assume you are right. Wait.

Never guess which of you is right. If the difference changes what the code
does, it is a logic divergence, however small.

When the last line of a section is in place, mark the section `[done]` and
move to the next pending one.

Other things the user may say:

- **next** with nothing typed: they want more. Explain the line again from a
  different angle, or justify why this line and not another. Do not skip.
- **skip**: move on without this line. Only on this explicit word.
- **slower**: split the current line into smaller statements.
- **why**: expand on the reasoning.
- **show me the flow**: restate where we are in the journey and what remains.

## Plan edits

There are two kinds, and they are not the same.

**Sync edit.** The user made a cosmetic change and you accepted it. Update
the plan so it matches the code, in that section and in every later section
that references the same name or construct. No confirmation needed. No
recalculation of the flow. The plan must mirror the code, or it lies to the
next session.

**Flow edit.** The user confirmed a logic change, or asked to rethink part of
the flow during a debate. Re-research the impacted methods and files, rewrite
the affected sections and any sections downstream of them, and show the user
the before and after of what changed. Only then continue. Never make a flow
edit without the user's explicit confirmation.

## Debating

The user may push back on anything: a name, a shape, an approach. Argue your
case if you have one, in a few lines. Concede when they are right or when it
is a matter of taste. Then decide whether the outcome is cosmetic or a change
in logic, make the matching plan edit, and continue.

## Resuming

If a plan already exists when the skill starts, read it. Open the first
section that is not done, read its file, and see which lines of its blocks
are already in place. Summarize the task, the journey so far, and where you
are. Then continue from the next line, after the user says they are ready.
