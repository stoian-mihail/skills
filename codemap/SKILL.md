---
name: codemap
description: >
  Map one area of the codebase as a flow-ordered markdown code map of the
  code as it exists today. Invoked explicitly with /codemap followed by a
  sentence describing the area. Do not activate on your own.
argument-hint: "<sentence describing the area to map>"
---

# Codemap

You draw the journey of the data through an area of the code, as it exists
today, into one markdown file. You read code; you never change it. The only
file you write is the map, at `.codemaps/<slug>.md`.

This is the map-only fraction of the `dont-rust` skill. Same tracing rules,
same depth, no plan and no dictation.

The argument is a sentence: how the user would describe the area to a
colleague. "The part where a test bank statement gets generated and stored."
"Whatever happens when the user downloads a certificate." Your first job is
to find what they mean. Your second is to confirm it. Only then do you map.

Two other forms:

- `/codemap refresh <slug or sentence>`: re-trace an existing map and report
  what moved, was added, was removed, or no longer resolves. Then rewrite it.
- `/codemap check <slug>`: verify every anchor in an existing map still
  points at the line it claims. Report; do not rewrite.

## Phase 1: Locate

Turn the sentence into search terms: the domain nouns, the verbs, and their
likely synonyms in code ("download" may be "export" or "stream"; "generate"
may be "build", "render", "create"). Search where entry points live:

- route paths and controller action names
- Vue components, and the API paths they call
- service and generator class names
- Twig template names
- console command names
- Messenger message and handler names

Read the candidates you find, enough to know what each one actually does.
Do not propose from file names alone. Rank them by how well what they do
matches the sentence.

## Phase 2: Propose

Show the user what you think they meant, as a short outline, not a map:
the entry point, the main hops in one line each, and where the flow ends.

"I think you mean this: the generate button in TestDocumentCreate.vue posts
to /test_documents/generate, the controller hands the fields to
TestDocumentGenerator, which routes to BankStatementGenerator, renders HTML,
converts it to PDF, and the controller persists the entity. Is that the
flow?"

- Several areas match: show each as a one-line candidate and ask which.
- Nothing matches: say so, and ask for a hint: a file, a URL path, a button
  label, a class name.
- The sentence names one flow but the module has siblings (bank statement
  next to identity card and certificate): map only the one asked for, and
  mention the siblings in a line so the user can ask for them. If the
  sentence names the module, map every flow in it.

The user answers. "Yes" means trace and write. Anything else means adjust
and propose again: a different flow, a wider or narrower scope, an earlier
entry point. Keep talking until they confirm. Never write a map before they
have confirmed the outline.

## Phase 3: Trace

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
entered it. Never assign letters in any other order.

When a call cannot be resolved statically, do not guess. Mark the node
unresolved, say what you found (the candidates, the count), and list it in
the Unresolved section.

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

## Phase 4: Write

Write `.codemaps/<slug>.md`. The slug is short and comes from the confirmed
outline, not from the sentence. Suggest committing the folder: a map is
documentation, and its header says how stale it is.

The file has:

1. A title, then one line: mapped at `<short commit hash>`, the date, and
   the entry points.
2. A summary paragraph of what the area does, referencing the flows by
   their letter ranges: "bank statement generation [1a–1f]".
3. One section per flow: a numbered heading, one line on where it starts
   and ends, then the tree as a nested list.
4. **Unresolved**: every node you could not follow, with what you found.
5. **Nearby flows not mapped**: the siblings, one line each, if any.

A key hop in the tree is one bullet: the bold letter and a verb-first label
the length of a commit subject, then a file link relative to the repository
root with the line as a fragment, so it opens in the editor and on GitHub.
Under it, indented: the prose paragraph (receives, does, hands on, branches),
then the lines that matter in a code block, as they are in the source.

A pass-through is one bullet in italics with the method name and nothing
else. Nesting follows the call structure.

Read `example-map.md` in this skill's folder for the shape and the depth.
It is a guideline, not a template: adapt names, depth, and the resolution
rules to the project in front of you.

When the file is written, tell the user where it is and repeat the
Unresolved list in chat. That list is where they can help.
