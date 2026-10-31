---
name: codemap
description: >
  Map a module of the codebase as a flow-ordered markdown code map of the
  code as it exists today. Invoked explicitly with /codemap followed by a
  sentence describing the area. Do not activate on your own.
argument-hint: "<sentence describing the area to map>"
---

# Codemap

You draw the journey of the data through a module, as it exists today, into
one markdown file. You read code; you never change it. The only file you
write is the map, at `.codemaps/<slug>.md`.

The unit of a map is the module. A flow is a path through it. A flow is never
alone: flows share an entry point and a dispatch, share subsystems, and one
flow writes what another reads. The map shows the flows and how they connect.

The argument is a sentence, how the user would describe the area to a
colleague. "The part where a test bank statement gets generated and stored."
"Whatever happens when the user downloads a certificate." Your first job is
to find what they mean, and the module it belongs to. Your second is to
confirm it. Only then do you map.

Two other forms:

- `/codemap refresh <slug or sentence>`: re-trace an existing map and report
  what moved, was added, was removed, or no longer resolves. Then rewrite it.
- `/codemap check <slug>`: verify every anchor in an existing map still
  points at the line it claims. Report; do not rewrite.

## Phase 1: Locate

Turn the sentence into search terms: the domain nouns, the verbs, and their
likely synonyms in code ("download" may be "export" or "stream"; "generate"
may be "build", "render", "create"). Search where entry points live: route
paths and controller action names, Vue components and the API paths they
call, service and generator class names, Twig template names, console
commands, Messenger messages.

Read the candidates you find, enough to know what each one actually does.
Do not propose from file names alone. Rank them by how well what they do
matches the sentence.

Then find the module around the flow you located. Ask, for that flow:

- what shares its entry point or its dispatch (the other arms of the same
  `match`, the other actions of the same controller, the other components
  on the same page)
- what subsystems it calls (pools, helpers, renderers used by several flows)
- what reads or writes the same data (the flow that lists or downloads what
  this one stores)

Each answer is a sibling flow or a subsystem flow. Together with the flow
you located, they are the module.

## Phase 2: Propose

Show the user what you think they meant, as a short outline, not a map: the
module, its flows in one line each, and which one they named.

"I think you mean the test document module. Flows: bank statement
generation (the one you named), identity card generation, certificate
generation, document listing and download, and the data pools they all draw
from. Shall I map the module?"

- Several modules match: show each as a one-line candidate and ask which.
- Nothing matches: say so, and ask for a hint: a file, a URL path, a button
  label, a class name.
- The user wants only the one flow: map only that one, and list the siblings
  under Nearby flows not mapped.

The user answers. "Yes" means trace and write. Anything else means adjust
and propose again. Keep talking until they confirm. Never write a map before
they have confirmed the outline.

## Phase 3: Trace

For each flow, follow the data from its entry point inward, depth-first,
until it comes to rest or leaves the system: a database write, a file on
disk, an HTTP response, an external call. If the response is then consumed
by the UI in a way that matters (a list refreshes, a download starts),
include those hops.

First enumerate. List every method the data passes through, in execution
order, by actually following each call. Only then classify:

**Key hop**: a method where the data is transformed, decided, stored, or
crosses a boundary (UI to HTTP, controller to service, service to database,
PHP to an external binary). Gets a letter, a label, an anchor, a paragraph,
and one line of code.

**Context node**: a human action that starts the flow ("User clicks
generate"), a boundary ("API endpoint receives"), or a method the data
crosses unchanged. Gets a line with its name and nothing else, so the path
stays unbroken.

**Branches become flows.** A `match` or `if` that sends the data down
different paths is not drawn as a branch inside a hop. Each arm is a flow of
its own. The hop that dispatches names the arms and which flow each one is.

**Shared parts are drawn once.** The first flow that uses a shared prefix
(Vue to controller to generator) or a shared suffix (the controller
persisting the result) gets it in full, with letters. Every later flow shows
the shared part as context nodes that name the hop where it was detailed:
"*TestDocumentController persists the entity (1f)*".

**Subsystems are flows.** A pool, a renderer, a helper used by several flows
is its own numbered flow, mapped once. A hop in a request flow that uses it
says so: "uses 5b".

**Data handoffs are stated at both ends.** "Stores the entity that 4b
lists." "Reads what 1f stored."

Letters follow execution order strictly, depth-first: 1a, 1b, 1c in the
order the code runs. A hop inside a method is nested under the hop that
entered it. The tree and the letters have one ordering, never two.

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

Write `.codemaps/<slug>.md`. The slug names the module, short, taken from
the confirmed outline. Suggest committing the folder: a map is
documentation, and its header says how stale it is.

The file, top to bottom:

1. **Title**: the module and the system. Then one line: created on, mapped
   at `<short commit hash>`.
2. **Abstract**: one paragraph. What the module is, what stack it runs
   on, what it produces. Then a flow index: a numbered list binding, in
   the same line, each flow's number, its name, a one-sentence
   definition, and its letter range:
   "- **Flow 1 — bank statement generation [1a–1f]**: the user picks a
   type, the generator renders the PDF, the controller stores the entity."
   Name and define at the same time, before anything references them: a
   reader must be able to learn what every flow is from this list alone,
   and no part of the map may use a flow's number before this index
   exists.
3. **Module**: how the flows connect, one line per connection: shared
   entry, shared dispatch, subsystems used, data handed from one flow to
   another. Re-bind the name on every reference — "flow 1 (bank
   statement generation)" — so each connection line stands alone.
4. **Flows**, one section each: a numbered heading, a one-line
   description, a collapsed block, then the tree.
5. **Unresolved**: every node you could not follow, with what you found.
6. **Nearby flows not mapped**: only when the user narrowed the scope.

The collapsed block is `<details><summary>See more</summary>` and holds
what belongs to the flow as a whole, not to one hop: its purpose, its
inputs and outputs, the error path, what it shares with other flows. It
collapses in Cursor's preview and on GitHub.

The tree is a nested list. A key hop is:

    - **1b** Controller calls the generator · [TestDocumentController.php:63](src/Controller/TestDocumentController.php#L63)
      Receives `type` and `fields` from the JSON body and checks only that both
      exist. Hands them to the generator and gets back a path relative to the
      documents directory. Builds the entity from it in 1f.
      `$relativePath = $this->generator->generate($type, $fields);`

Bold letter, then a verb-first label the length of a commit subject, then
the anchor: a link relative to the repository root with the line as a
fragment, so it opens in the editor and on GitHub. Under it, the paragraph:
two to four sentences, what the hop receives, what it does with it, what it
hands on, and the branch or the cross-reference if there is one. Then one
line of code, the representative statement, as it is in the source. One
line: the paragraph carries the understanding, the line anchors it.

A context node is one bullet in italics with its name and nothing else.
Nesting follows the call structure.

Skim the bold lines and you have the shape. Read the paragraphs and you
have the journey. Nobody has to click.

Read `example-map.md` in this skill's folder for the shape and the depth,
at module scale. It is a guideline, not a template: adapt names, stack, and
the resolution rules to the project in front of you.

When the file is written, tell the user where it is and repeat the
Unresolved list in chat. That list is where they can help.
