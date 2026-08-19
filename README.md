> This is experimental software. It probably doesn't work.

# coil-agent-gui

A three-pane native window for the Coil agent harness, written in Coil against
AppKit and CoreGraphics with no third-party dependency. The whole binary is
under 200 KB and opens a window in the time it takes to create one.

```sh
coil run                    # the window, backed by the harness on disk
coil run -- --real          # the same, rendered headless to real.png
coil run -- --render        # five design scenes, no window, scripted data
coil run -- --spawn-test    # prove the spawn path without starting an agent
coil run -- --click X Y     # drive a click headlessly and print what changed
coil run -- --author "…"    # author a workflow headlessly and print what landed

`--real` takes an optional width, height, and scroll-back distance, which is how
the layout is checked at sizes other than the one it opens at.
```

`COIL_HARNESS_ROOT` points at the harness checkout; it defaults to the sibling
`../coil-agent-harness`.

Run from the project root; the fonts resolve from `assets/fonts`.

## The idea

Left: what exists — projects, and the workflows under them. Middle: the shape of
the one you picked. Right: the one conversation you are reading, with a composer
that talks to it.

There is no create mode. `cmd N` opens an empty conversation; you describe what
you want and the middle pane fills in as you talk. A plan is just a graph whose
steps are all queued, so nothing new appears on screen when one exists — the
same nodes simply have a different state. The screen you make a workflow on is
the screen you run it on and the screen you watch it from.

## What is here

- Three panes, a rail that groups workflows under projects, a dependency graph
  laid out from the dependencies themselves, and a conversation view.
- Six step states — queued, running, done, failed, cancelled, and **asks**, the
  one that means a step is waiting on a person. Asks is amber, on the node and
  on the rail row, and it is answered in the conversation rather than on a
  separate approvals screen.
- Talking changes the graph: `add <step>` and `drop <step>` restructure the plan
  mid-conversation and the layout follows without anything being placed by hand.
- A run that advances step by step, stops at a step running on another machine,
  and carries on when you answer.
- Text that elides rather than clipping, one accent colour with one meaning, and
  hairline separation instead of panels.

## What is real

The window is backed by the harness on disk, not by fixtures:

- **Workflows are factories.** `factories/<folder>/factory.json` gives each one
  its name and its workers, and the workers are the steps in the graph.
- **State is the journal.** `.factory-runs/<name>/<timestamp>.jsonl` holds the
  durable event records a run wrote — the same versioned schema the harness's own
  TUI replays. Step state comes from `run.started` / `run.completed`, and the
  rail's count is the factory's own `factory.stage.accepted` records.
- **Conversations are replayed**, not summarised: assistant prose is rebuilt from
  `model.response.delta`, and every tool call and result is the one that ran.
  Selecting a different step replays that step out of the journal.
- **Run starts a run.** It shells out to `./harness factory run <folder>` in the
  harness directory, exactly as a person would, then watches for the journal the
  run writes and reloads as it grows.
- **Typing writes a workflow.** In an empty conversation (⌘N), what you type
  becomes a real one-worker factory on disk — a manifest and a Markdown worker,
  the same shape a person would write. It is not started until you press Run, so
  typing never spends anything. The manifest carries an explicit empty
  `context`, because the harness's loader rejects a manifest without one.

## What is not here

- **You cannot talk to a run.** A recorded run has nobody to answer, and a live
  one cannot be reached either: the harness's only intervention is `cancel`.
  Typing into an open workflow says so rather than pretending. Conversation with
  a running agent needs a `message` intervention that does not exist yet.
- **Projects are still one label.** Every factory is filed under "harness". Real
  projects — separate checkouts, with workflows that can run against any of
  them — are the next piece.
- The design scenes behind `--render` are still fixtures. They are a design
  tool, not a view of anything.
- The window resizes freely; panes are derived from its size, and the
  conversation scrolls with the wheel and is anchored to its end.
- No text selection, no keyboard scrolling, and ⌘K does nothing yet.
- macOS only.

## Layout

- `src/ffi.coil` — every C declaration in one module, because Coil does not
  dedupe `extern` across modules.
- `src/theme.coil` — one ground, two hairlines, three inks, three signals.
- `src/draw.coil` — the CoreGraphics backend: the only file that knows the y
  axis points the other way, and the only one that touches CoreText.
- `src/marks.coil` — the state marks, drawn as geometry rather than typed. A
  terminal has to spell these with whatever glyphs the font carries; a window
  does not.
- `src/model.coil` — steps, conversation, the planner seam, the run engine.
- `src/view.coil` — the three panes, and the hit testing that mirrors them.
- `src/harness.coil` — the real backing: factories, journals, replay, and the
  one call that starts a run.
- `src/dir.coil` — directory listing, adapted from the harness's own, because
  `coil.fs` reaches files and not directories.
- `src/main.coil` — the window, the offscreen renderers, and the scripted run.

Fonts are IBM Plex Mono under the SIL Open Font License; see `IBM-Plex-OFL.txt`.
