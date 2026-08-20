> This is experimental software. It probably doesn't work.

# coil-agent-gui

A three-pane native window for the Coil agent harness, written in Coil against
AppKit and CoreGraphics with no third-party dependency. The whole binary is
under 200 KB and opens a window in the time it takes to create one.

```sh
coil run                    # the window, backed by the harness on disk
coil run -- --real          # the same, rendered headless to real.png
coil run -- --shot KIND N   # the same, with a project/workflow/run/issue open
coil run -- --spawn-test    # prove the spawn path without starting an agent
coil run -- --click X Y     # drive a click headlessly and print what changed
coil run -- --rail          # print the rail: projects, runs, issues, library
coil run -- --run N         # print the command Run would use for workflow N
coil run -- --run-now N     # actually run workflow N here and follow it
coil run -- --say N "…"     # run workflow N, then talk to it while it runs
coil run -- --author "…"    # ⌘N end to end: an agent designs and writes it
coil run -- --select x y X Y  # drag across the conversation and print what copies
coil run -- --keys          # every keystroke this app handles, including ⌘C
coil run -- --mouse         # press, drag, release across the conversation
coil run -- --stop N        # run workflow N, then click Stop and prove it died
coil run -- --run-issue N   # run issue N through its workflow and follow it
coil run -- --backlog "…"   # talk to the backlog agent and print what it wrote

`--real` takes an optional width, height, and scroll-back distance, which is how
the layout is checked at sizes other than the one it opens at.
```

`COIL_HARNESS_ROOT` points at the harness checkout; it defaults to the sibling
`../coil-agent-harness`.

Run from the project root; the fonts resolve from `assets/fonts`.

## The idea

Three nouns, and the window is built on the difference between them:

- a **project** is a directory you have — a checkout, declared once in
  `~/.coil-agent-harness/projects.json`;
- a **workflow** is a reusable definition. It belongs to no project;
- a **run** is one workflow, executed once, in one project. It is the only one of
  the three that has any state.

So the left rail is projects, and under each one what has happened there: its
runs, newest first, and the issues filed in it. Selecting a project fills the
middle pane with the library under one question — run a workflow here — and
picking one puts both halves on the button: `Run in coil-agent-harness`.

Running and creating are different acts, and both of them are conversations. ⌘N
talks to an agent that designs a workflow and writes it into the library; it
never asks for a project, because a workflow does not have one. ⌘I talks to an
agent that keeps the backlog of the project you are looking at, and writes issues
into it. Neither of them starts a run.

## What is here

- Three panes: what exists, what you can do here, and what was said.
- A dependency graph laid out from the dependencies themselves, six step states,
  and a conversation replayed from the journal a run wrote.
- Text that elides rather than clipping, one accent colour with one meaning, and
  hairline separation instead of panels.

## What is real

Every row comes off disk. There are no fixtures anywhere in this app.

- **Projects are `projects.json`** — the same file `harness projects` reads.
- **Workflows are factories.** `factories/<folder>/factory.json` gives each one
  its name and its workers, and the workers are the steps in the graph. Nothing
  in a manifest names a project.
- **Runs are journals.** `.factory-runs/<workflow>/<timestamp>.jsonl` holds the
  durable records a run wrote, and the **workspace that journal recorded** is what
  files the run under a project. A run also records its own pid, so one whose
  process is gone without an ending — killed, crashed, laptop closed — reads as
  `abandoned` instead of spinning in the rail forever. A run in a scratch directory belongs to no
  project and is not shown, because the rail is projects and a scratch directory
  is not one.
- **Conversations are replayed**, not summarised: assistant prose is rebuilt from
  `model.response.delta`, and every tool call and result is the one that ran.
- **Run starts a run.** `./harness factory run <folder> --project <name>` for a
  workflow, `./harness factory issue <workflow> --issue <file> --project <name>`
  for an issue — the same commands from a terminal. The window then watches for
  the journal it writes and follows it as it grows.
- **⌘I is a conversation with the backlog.** You talk about what is wrong; an
  agent reads the project — it can reach the checkout with bash — asks about what
  is ambiguous, and writes the issues as files under `.factory-issues`, one per
  piece of work, with the project in their front matter. One paragraph about
  three problems becomes three issues. Keep talking and it revises them. Nothing
  runs until you pick one and press Run.
- **A worker that says it is not ready does not end the run.** The harness hands
  the worker its own account of what remains and runs it again, in the same
  workspace, up to five attempts. The rail shows the attempt; the run carries on
  to the next step when it is genuinely done.
- **You can talk to a run after it has finished.** Typing at a finished run starts
  the same workflow again, in the same project, with what you said as its
  instruction and the finished run named so the agent picks up rather than
  starting over. The window switches to watching the new run.
- **You can talk to a run while it is running.** Open a live run and the composer
  sends into it: the message reaches the agent at its next turn, it acts on it
  there, and it stays in force for the rest of the run. What you said appears in
  the transcript where you said it, because that is where the journal has it.
- **A workflow that needs an issue cannot be run without one.** Its manifest says
  `"input": "issue"`, so it has no Run of its own; the middle pane asks which
  issue to run it on, and the CLI refuses the same way.
- **⌘N is an agent.** What you type goes to a real agent through the harness's
  own service — started on demand, on loopback, on a port derived from this
  checkout so a service started for another one is never reused (its file tools
  are rooted where it was started, and sharing a port would write one project's
  workflows into another) — via `POST /v1/runs` and its event stream, which decides the steps,
  writes `factory.json` and one Markdown worker per step with its file tools, and
  says what it made. The graph fills in from the folder it wrote. Keep talking and
  it edits the same workflow. `COIL_GUI_PROVIDER` and `COIL_GUI_MODEL` choose who
  that agent is; it defaults to the Claude subscription.
- **Text is selectable.** Drag across the conversation, ⌘C copies, ⌘A takes all of
  it. Selection is character-exact because every face here is fixed-pitch.
- **The composer wraps and grows.** It holds a paragraph: text wraps at the pane's
  measure instead of running off the right, ⇧⏎ starts a new line, ⏎ sends, and the
  strip grows upward while the conversation moves up to make room.
- **Stop is beside the transcript**, in the right-hand pane's header — where you
  are looking when you decide to stop something.
- **A failed start is visible.** What the harness printed when it refused to start
  goes in the pane, instead of the window quietly waiting forever.

## What is not here

- **Stop kills.** A run this window started can be stopped from its header: SIGTERM
  to the harness process itself, and a marker beside the journal, because a killed
  run is not around to record its own ending. A run somebody else started is
  watchable but not stoppable — you can still tell it to stop, which is a
  conversation rather than a kill.
- **A message is not an interrupt.** It lands at the next turn boundary, so if the
  model has just queued ten tool calls it lands after those.
- **You cannot talk to a run.** A recorded run has nobody to answer, and a live
  one cannot be reached: the harness's only intervention is `cancel`. A `message`
  intervention does not exist yet, which is also why the amber "asks" state has
  nothing to produce it.
- **Running a workflow in a real checkout leaves a `.factory/` directory in it**,
  holding the run's private copy of the workflow. Everything a run brings with it
  lives there now; it used to drop a `context.md` at the top of your repository.
- No text selection, no keyboard scrolling, and ⌘K does nothing yet.
- macOS only.

## Checking it

`coil test` covers the parts where being wrong is quiet: which project a run
belongs to, front matter, where an issue's body starts, slugs, escaping.

Everything else is checked by driving the app headlessly through the same
functions the window calls — `--keys` types, `--mouse` drags, `--stop` clicks the
button, `--run-issue` runs a real agent end to end. They are not a substitute for
using it, but a click that stopped working has been caught by them twice.

## Layout

- `src/ffi.coil` — every C declaration in one module, because Coil does not
  dedupe `extern` across modules.
- `src/theme.coil` — one ground, two hairlines, three inks, three signals.
- `src/draw.coil` — the CoreGraphics backend: the only file that knows the y
  axis points the other way, and the only one that touches CoreText.
- `src/marks.coil` — the state marks, drawn as geometry rather than typed. A
  terminal has to spell these with whatever glyphs the font carries; a window
  does not.
- `src/model.coil` — steps, conversation, and what is open in the panes.
- `src/view.coil` — the three panes, and the hit testing that mirrors them.
- `src/harness.coil` — the real backing: projects, workflows, runs, issues,
  journal replay, and the two calls that start work.
- `src/dir.coil` — directory listing, adapted from the harness's own, because
  `coil.fs` reaches files and not directories.
- `src/main.coil` — the window, the offscreen renderers, and the scripted run.

Fonts are IBM Plex Mono under the SIL Open Font License; see `IBM-Plex-OFL.txt`.
