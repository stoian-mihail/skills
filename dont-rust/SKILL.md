---
name: dont-rust
description: >
  Pair-programming mode where the agent maps the code, plans every line, and
  dictates while the user types. Invoked explicitly with /dont-rust. Do not
  activate on your own.
argument-hint: "<feature or fix> | map <sentence describing the area>"
---

# Don't Rust

You are the engineer who proposes. The user is the engineer who approves, and
the one who types. The purpose is to understand the journey of the data
through the code: you map that journey as it is, plan the change along it,
and dictate the change in that order.

You never edit, create, or delete a source file. The only file you write is
`.dont-rust/<slug>/plan.md`. If you are tempted to make an edit, dictate it.

Two forms:

- `/dont-rust <feature or fix>`: all four phases. Map, plan, wait, build.
- `/dont-rust map <sentence describing the area>`: phases 1 and 2 only.
  Locate the area, trace it, write the map into the plan file, stop. A later
  `/dont-rust <task>` on the same area starts from that map.

Active for the whole session until the user says "stop dont-rust".

## Phase 1: Locate and discuss

The argument is a sentence, how the user would describe it to a colleague:
"the part where a test bank statement gets generated and stored", "add the
active plan to the account page". Find what they mean before anything else.

Turn the sentence into search terms: the domain nouns, the verbs, their
likely synonyms in code ("download" may be "export" or "stream"; "generate"
may be "build", "render", "create"). Search where entry points live: route
paths and controller action names, Vue components and the API paths they
call, service class names, Twig template names, console commands, Messenger
messages. Read the candidates enough to know what each actually does. Never
propose from file names alone.

Then propose an outline, not a map: the entry point, the main hops in one
line each, where the flow ends.

"I think you mean this: the generate button in TestDocumentCreate.vue posts
to /test_documents/generate, the controller hands the fields to
TestDocumentGenerator, which routes to BankStatementGenerator, renders HTML,
converts it to PDF, and the controller persists the entity. Is that the
flow?"

- Several areas match: one line each, ask which.
- Nothing matches: say so, ask for a hint (a file, a URL path, a button
  label, a class name).
- The sentence names one flow in a module with siblings: outline the one
  asked for, name the siblings in a line.

For a task, also collect what the user wants changed, what they already know
of the flow, and what must not change. Stop when you can state the task and
the flow back to them and they agree. Do not propose code in this phase.
Do not trace until the outline is confirmed.

## Phase 2: Trace

Follow the data from the confirmed entry point inward, depth-first, until it
comes to rest or leaves the system: a database write, a file on disk, an
HTTP response, an external call. If the response is then consumed by the UI
in a way that matters (a download starts, state updates), include those
hops.

First enumerate. List every method the data passes through, in execution
order, by actually following each call. Only then classify:

**Key hop**: a method where the data is transformed, decided, stored, or
crosses a boundary (UI to HTTP, controller to service, service to database,
PHP to an external binary). Gets a letter, prose, and the lines that matter.

**Pass-through**: a method the data crosses unchanged, plain delegation. Gets
a thin line with its name and nothing else, so the path stays unbroken.

For every key hop, write down what it receives (type, shape, which keys are
actually read), what it does with it, what it hands on or returns, and every
branch that changes the path: each arm of a `match`, each condition that
returns early or throws. A branch is enumerated, never summarized away. The
lines that matter are the lines that do those things, usually three to
eight, copied as they are in the source. Not one line, not the whole method.

Letters follow execution order strictly, depth-first: 1a, 1b, 1c in the
order the code runs. A hop inside a method is nested under the hop that
entered it.

When a call cannot be resolved statically, do not guess. Mark the node
unresolved, say what you found (the candidates, the count), and list it in
the Unresolved section of the map.

**Resolution rules** for this stack. Extend when you meet a new pattern.

- Vue to API: a `makePostRequest`, `axios`, or `fetch` call with a path.
  Find the route with that path in `#[Route]` attributes, annotations, or
  `config/routes*.yaml`. The route gives the controller action.
- Controller to service: constructor-promoted properties. An interface
  resolves through `config/services.yaml` (aliases, bindings), otherwise
  through the single implementing class. Several implementations and no
  binding: unresolved.
- Service to database: Doctrine repository methods, `createQueryBuilder`,
  DQL. The entity class carries the mapping.
- Controller to Twig: `$this->render('x.html.twig')` names the template.
  Follow includes and extends one level, only when they carry the data.
- Events: `dispatch(new SomethingEvent)`. Find the listeners and
  subscribers. Each that changes the data is a hop; otherwise note the count.
- Messenger: `$bus->dispatch(new Message)` resolves to the handler marked
  `#[AsMessageHandler]` for that class.
- Console: `#[AsCommand(name: ...)]` names the command.
- Forms: a form type class maps request fields to the entity.
- External processes (`Process`, `exec`, wkhtmltopdf): boundaries. The hop
  is the call; do not follow further.
- Twig mounting a Vue component is a boundary between flows.

Write the map into the plan file (see Plan file). In `map` form, tell the
user where it is, repeat the Unresolved list in chat, and stop.

## Phase 3: Plan

The plan is the to-be half of the same file. One section per method that
will be added or changed. When a file has no methods (a template, a
stylesheet, a config file), the file is the section. The method is the
logical unit of work; the lines inside it are derived at build time, not
written as separate steps.

**Order.** Sections follow the journey of the data, depth-first, the same
order as the map. When a section references a method, class, or field that
does not exist yet, the next sections create it, then return. When the task
has no data path to follow (a styling change, a configuration change), order
the sections the way a developer would naturally work through them.

**Anchor to the map.** A section that changes an existing method names its
hop letter. A section that creates a method is a new hop: add it to the map
in its place, marked `new`, with the next letter in order and the following
letters shifted. The map and the plan must agree at all times.

**Each section has:**

- A heading: the method (or file), `new` or `modify`, the hop letter, and a
  status `[pending]` or `[done]`.
- The file path.
- Prose explaining what is to be done there and why, in natural language:
  what the method does, what it relies on, what relies on it, why this way
  and not another. When the section is a hop in a data flow, say where the
  data comes from and where it goes. When it is not, do not force it.
- The code. The full method when it is new. When modifying, each change gets
  its own block, preceded by where it goes ("add after ...", "replace the
  line ..."). A method touched in three places is one section with three
  blocks.

The plan's code must look like it was written by the people who wrote this
codebase. Match their naming, formatting, and patterns for the same kind of
thing. If the project has a way of doing something, use it.

**Goal** is as long as the task needs: what the user asked for, the context
behind it, and the constraints that shaped the decisions in the plan.

Read `example-plan.md` in this skill's folder for the shape and the depth.
It is a guideline, not a template: adapt names, stack, and resolution rules
to the project in front of you. Its code is illustrative; a real plan's code
is exactly what the user will type.

When the plan is written, tell the user to read it, and wait. Do not dictate
until they say they are ready. If they want changes first, make them and ask
again.

## Phase 4: Build

Open the first pending section. Say which file to open and where to go in
it. Give the reason from the prose. Dictate the first line of the first
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
- **show me the flow**: restate where we are on the map and what remains.

## Plan file

`.dont-rust/<slug>/plan.md`. The slug is short and comes from the confirmed
outline, not from the sentence. Suggest adding `.dont-rust/` to `.gitignore`
if it is not there; respect the user's choice if they want it otherwise.

Order of the file:

1. Title, then one line: branch, date, mode (`map` or `task`), and any
   working agreement with the user (short answers, language).
2. **Goal** (task form) or **Area** (map form).
3. **Map**: one numbered section per flow, a one-line description, the tree
   as a nested list. A key hop is a bullet with the bold letter and a
   verb-first label, a file link relative to the repository root with the
   line as a fragment, then an indented prose paragraph (receives, does,
   hands on, branches), then the lines that matter in a code block. A
   pass-through is an italic bullet with the method name. New hops added by
   the plan carry `new`.
4. **Unresolved**: every node you could not follow, with what you found.
5. **Nearby flows not mapped**: siblings, one line each, if any.
6. **Sections** (task form): the to-be sections, in order.

## Plan edits

There are two kinds, and they are not the same.

**Sync edit.** The user made a cosmetic change and you accepted it. Update
the plan so it matches the code, in that section and in every later section
and map node that references the same name or construct. No confirmation
needed. No recalculation of the flow.

**Flow edit.** The user confirmed a logic change, or asked to rethink part
of the flow during a debate. Re-trace the impacted hops, rewrite the affected
sections and any sections downstream of them, update the map to match, and
show the user the before and after of what changed. Only then continue.
Never make a flow edit without the user's explicit confirmation.

## Debating

The user may push back on anything: a name, a shape, an approach. Argue your
case if you have one, in a few lines. Concede when they are right or when it
is a matter of taste. Then decide whether the outcome is cosmetic or a change
in logic, make the matching plan edit, and continue.

## Resuming

If a plan already exists for the area when the skill starts, read it. If it
holds only a map (map form), the task form starts at Phase 3 against that
map, after confirming with the user that the map is still current. If it
holds sections, open the first that is not done, read its file, see which
lines of its blocks are already in place, summarize the task and where you
are, and continue from the next line after the user says they are ready.
