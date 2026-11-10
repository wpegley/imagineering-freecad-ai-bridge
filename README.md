# Imagineering AI Bridge

A FreeCAD macro that lets an AI agent drive your live CAD session over
plain files on disk. No socket, no API, no vendor lock-in — an agent
appends a line of JSON to a file, FreeCAD executes it on the GUI thread,
and the result comes back as a line.

Anything that can write to a text file can drive FreeCAD: a coding
assistant, a local Ollama model, a shell script.

Tested on FreeCAD 26.3. Single file, no dependencies beyond what FreeCAD
already ships.

---

## Why it exists

> To make a FreeCAD document, and any AI connection, able to pick up what
> geometry exists and how that geometry was created — so that even after
> closing, saving and reopening, the AI can continue.

That is the whole point, and it is the thing to judge it on. Open a
`.FCStd` this macro has logged and the file carries its own account:
what the model is, what drives it, and what was done to it. Nothing
beside the file is needed.

## Parametric, not positional

The Bridge reads the model's **spreadsheet aliases and expression
bindings** and hands them to the agent before it writes a line of code,
together with the warning that hand-editing a bound property is
overwritten on the next recompute.

So the agent changes the number the part was designed around, rather
than pushing the geometry that number produced. Your design intent
survives contact with it.

## What it does

**Drives FreeCAD.** Commands execute in the session you are looking at,
sub-second round trip, with crash recovery and a watchdog.

**Describes the model.** Automatic inventory: volumes, centroids,
extents, materials, mass, what is parametric, what is already broken —
so an agent that cannot see the model does not put a 0.5 mm fillet on a
bridge. It refuses to guess: no centre of gravity unless every solid has
a stated density, no mass from a unitless number, and FreeCAD's default
"air" material card is rejected by UUID, by name and by a plausibility
band.

**Shows the model.** Visual Capture slices along X, Y and Z in a *second
FreeCAD process*, so your session never stalls. Each image carries the
true section outline at that plane in white, with every other plane's
outline behind it in cyan and ahead in amber — a contour map of the
shape, not just a cutaway. Fidelity is chosen in engineering decades
from 0.001 mm to 1 km, driving both the mesh and the grid, so a section
can be measured straight off the image.

**Logs the project.** A per-document Markdown log beside the `.FCStd`,
and a copy embedded *inside* it on every save, so the account travels
with the file.

**Outsources mesh work.** A registry of tools better at meshes than
FreeCAD is — push geometry out, work on it there, pull the result back.
Each target keeps four independent locations: data out, data in, its
agent training document and its agent database. The first is
[ScanRuler](https://github.com/CNCKitchen/ScanRuler) by Stefan Hermann
of CNC Kitchen, for turning a scan mesh into a solid STEP body. It is
driven as a separate program and never embedded, because it is AGPL-3.0.

**Protects your work.** A standing notice on every tab, and one button
that saves a dated copy into a `Forks` folder beside the document —
`saveCopy`, so you stay in the file you were in. The agent is told when
a backup is due, so raising it is its job rather than your memory.

## The agent API

Everything above is reachable by the agent, not just by the person at the
panel. `AIBridge` is pre-loaded in every `execute` command; a `run_macro`
file reaches the same object as `App.AIBridge`.

```python
AIBridge.memory_recall()        # what this document already knows
AIBridge.memory_start()         # begin logging (save the document once first)
AIBridge.memory_note('why')     # record a decision in the log
AIBridge.memory_checkpoint()    # flush, and re-embed into the .FCStd
AIBridge.memory_stop()          # embed and stop
AIBridge.inventory()            # extent, volume, centroid, what is parametric
job = AIBridge.capture()        # images and sections, in a SECOND process
AIBridge.capture_status(job)    # the manifest once that process is done
print(AIBridge.help())          # the full list
```

The memory is embedded in the `.FCStd` itself, so a document opened cold
on another machine still answers `memory_recall()`.

## Install

Copy `001_AI_Bridge_5.0.19.FCMacro` into your FreeCAD macro directory
and run it. The I/O directory is created under FreeCAD's own per-version
app-data path; nothing is written outside it unless you point the
Locations tab somewhere else.

## Talking to it

Register first, then send commands:

```json
{"id": "hello_1", "type": "hello", "agent_id": "my-agent"}
```

The reply is a `registration_ack` carrying a `context_hash`. Then:

```json
{"id": "cmd_1", "type": "execute", "agent_id": "my-agent",
 "command": "doc = App.ActiveDocument\nimport Part\n..."}
```

Each command runs in a **fresh scope** — nothing persists between them,
so every command re-imports what it needs and starts with
`doc = App.ActiveDocument`. Read `ai_output_results.jsonl` back and
confirm you got a result; appending and walking away proves nothing.

## What it does not claim

**The `execute` sandbox is a typo guard, not a security boundary.** It
is a textual blacklist, an AST import check and a deny-list of builtins
(`open`, `eval`, `exec`, `compile`, `globals`, `setattr` and a few more),
and it can be walked around. `run_macro` has no sandbox at all, by design. Treat this macro
as what it is: something that runs code you or your agent send, on your
machine, under your eye. Do not expose it to anything you would not hand
a shell to.

**Windows only, today.** The core is portable — paths go through
`os.path`, the I/O directory comes from FreeCAD itself, and FreeCADCmd
is resolved per platform — but editor detection ships Windows paths
only, and the optional text-to-speech shells out to PowerShell. It has
not been run on Linux or macOS.

**A paused Bridge looks healthy from outside.** The status file reports
`monitoring_active: true` with a fresh heartbeat while paused. Known,
not yet fixed.

**There is no command timeout.** A runaway `execute` holds the GUI
thread until it finishes. The watchdog reports the freeze; it cannot
stop it.

## Changes

**5.0.19** — Editor launching. Zed was handed VS Code's `--reuse-window`,
refused it, and opened nothing; Claude Code's terminal failed its `cd` on
cmd.exe quoting and never started `claude`. Both were reported as
"launched", because starting a process is not the editor opening — a
launcher that exits with an error is now reported with what it said.
Antigravity is now detected. The connection instructions written for an
editor's AI left `agent_id` and `context_hash` off the command line it was
told to copy, so following them exactly was rejected; they now carry both.
Tested on this machine: VS Code, Cursor, Devin, Zed, Antigravity and
Claude Code all open on the workspace.

**5.0.18** — The first Part object an agent created could crash FreeCAD.
The sandbox gave commands a 30-name allow-list of builtins, and the first
Part object loads PartGui *inside* the command, under those builtins. Its
start-up code failed inside C++, and C++ aborted: `InterruptedError: Break
signal occurred` the first time, a dead FreeCAD the next, or a silent
0%-CPU freeze. The sandbox now removes a deny-list instead, which also
gives agents exception classes, `repr`, `map` and `next`.

**5.0.17** — The memory and multicore features existed but no agent could
call them; they are now the `AIBridge` API above. Errors carry their real
traceback (every one used to read `NoneType: None`). A frozen GUI thread
now leaves its stack in `ai_stack_dump.txt`, so a freeze says where.
A restart no longer inherits the last session's `timers_dead` status.

Verified live on FreeCAD 26.3: Part object creation, the full memory
cycle through close and reopen, the inventory figures, capture in a
second process, and the Bridge's 32 self-tests.

## Credits

Built with [Claude Code](https://claude.com/claude-code).

ScanRuler is by Stefan Hermann (CNC Kitchen), AGPL-3.0, used as a
separate program.

## Licence

MIT — see [LICENSE](LICENSE).
