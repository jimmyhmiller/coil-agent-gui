> This is experimental software. It probably doesn't work.

# coil-agent-gui

A three-pane native window for the Coil agent harness, written in Coil against
AppKit and CoreGraphics with no third-party dependency. The whole binary is
under 200 KB and opens a window in the time it takes to create one.

```sh
coil run                 # the window
coil run -- --render     # five scene screenshots, no window
coil run -- --demo       # drive the real state machine from a script, render it
```

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

## What is not here

- **The planner is scripted.** `planner-respond!` in `src/model.coil` answers
  from the text you typed rather than from a model. Everything downstream works
  off the step list, so a real provider call replaces that one function.
- **Nothing is connected to the harness yet.** The next seam is
  `service-handle-http` in the harness, which is transport-neutral by design:
  local work is a direct call, and a step on another machine is the same call
  over HTTP.
- No scrolling, no text selection, no resize. The window is a fixed 1440×900.
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
- `src/main.coil` — the window, the offscreen renderer, and the scripted run.

Fonts are IBM Plex Mono under the SIL Open Font License; see `IBM-Plex-OFL.txt`.
