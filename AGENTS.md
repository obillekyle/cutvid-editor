# cutvid: driving the editor from a script

This page exposes one namespace for agents and scripts: **`window.editorAgent`**.
Open the app, then in the page:

```js
editorAgent.help()            // the reference, printed and returned
editorAgent.help('commands')  // topics: commands, state, media, text, program, ui, frames, session, export
```

Everything the interface does goes through one command bus, and so can a script:

```js
editorAgent.describe()                                  // every command, its params, its scope
editorAgent.dispatch([{ name: 'clip.insert', params }], { actor: 'agent:me' })
editorAgent.state()                                     // the document as JSON
editorAgent.sequences(); editorAgent.assets(); editorAgent.openSequence(id)
// asset kinds: video | audio | image | sequence | lut. A .cube imports as a LUT; segment.setLut { lut: hash } applies it
await editorAgent.importFiles([file]); await editorAgent.importUrls(['/x.mp4'])
await editorAgent.transcribe(hash); await editorAgent.readScreen(hash)
editorAgent.jobs()                                      // the Activity dock as data: title, detail, pct, waiting, ended
//   one decode-heavy job at a time: a transcription, a scan or an export asked for while another runs
//   waits its turn (jobs() shows it 'waiting' behind the other) and starts the moment the other ends
editorAgent.findText('pricing')                         // answers in timeline frames
editorAgent.program.seek(f); editorAgent.program.play(); editorAgent.program.snapshot()
editorAgent.source.open(hash); editorAgent.playback.rate(0.5); editorAgent.playback.proxies(false)
editorAgent.playback.hevc('software')       // or 'gpu': an HEVC file is decoded once at import and kept as H.264
editorAgent.ui.action('razor'); editorAgent.ui.select({ track: 1, index: 3 }); editorAgent.ui.tool('hand')
// every clip, image and nest wears rounded corners: segment.setTransform { radius } in px on the 1080 reference, 12 by default, 0 for square
// transitions: segment.setTransition { kind, durF, dir, soft, tiles, color, ease, blend, asset, seed } with 34 kinds the renderer draws, 32 of them in the picker
//   (dissolve, dip, slide, push, wipe, checker, checkerwipe, tiles, blinds, clock, iris, star, barn, softwipe,
//   bandslide, pinwheel, zigzag, dipcolor, flash, crosszoom, spin, flip, squeeze, shake, additive, slideease,
//   pushease, leak, burn, footage, whip, zoomblur, pixelate, glitch); ?/test/transitions on this origin shows each moving
//   a nested sequence takes one like any clip, on it and into it: what arrives is the sequence's own
//   picture at its first frame, composed in its own frame and placed by its own transform. Transitions
//   inside a nested sequence play wherever that sequence is used, and the file cuts the way the monitor does.
// a title's animation: title.set { animIn, animInF, animDuring, animPeriodF, animOut, animOutF, animUnit, animStagger, animColor }
//   kinds are the ids at ?/test/transitions-text (fade, rise, scribble, boil, dust…); '' for none; lengths in frames
// a clip's animation, on its transform and opacity: segment.setAnim { in, inF, during, periodF, out, outF } on a clip, an image or a nest
//   an in or an out is one of fade, zoom-in, zoom-out, slide-left, slide-right, slide-up, slide-down, rise, drop, pop, twirl, shake, blink
//   a during is one of breathe, pulse, sway, bob, drift, pan, wiggle, tremble, flicker, spin; '' for none; lengths in frames
// a layer's effect stack, whole: segment.setFx { fx: [{ kind: 'drop'|'inner'|'blur', x, y, blur, spread, color, opacity, on }] }
//   any visual segment; applied in list order; [] takes it off; sizes are 1080-reference px
// a clip's audio chain, in one command: segment.setAudioFx { denoise, hp, low, mid, midHz, high,
//   gate, comp, ratio, echo, echoMs, echoFb, room, roomSize, deEss, deEssHz, notchHz, notchQ,
//   notchDb }. Gate and comp are OFF by absence (null removes one, clear: true removes the
//   lot), and the chain runs denoise, low cut, EQ, notch, de-esser, gate, leveller, echo, room. It applies to a processed copy of the SOURCE, so a loop, a
//   reverse (segment.setSpeed with a negative speed) and a speed all read it unchanged.
await editorAgent.frames(asset, 100, 160)               // exact frames on one canvas, captioned, for a vision model
await editorAgent.export('cut1', { width: 1280, height: 720, mode: 'full' })
const s = editorAgent.session('my-tool', { scopes: ['edit-timeline'] }); s.open()
```

Rules worth knowing before the first batch:

- A batch is one transaction and one undo unit; it applies whole or not at all.
- Timeline positions are derived, never stored: a segment's place is the sum of the
  durations before it on its track. Never write a position field.
- A batch from an `agent:` actor never enters the human's undo. Take a checkpoint
  (`editorAgent.checkpoint(name)`) or open a session and revert through that.
- `dryRun: true` applies to a fork and returns the state it would produce.

This page is the whole of the public reference. The repository is private, so
nothing here points into it. What is always reachable is the app itself:
`editorAgent.help()` for the prose, `editorAgent.describe()` for every command
with its parameters and scope, both generated from the code rather than written
beside it.

And the procedures ship with the app, for the work rather than the API:

```js
await editorAgent.skills()                     // what this app knows how to do
await editorAgent.skill('slides-and-speech')   // the document, as text
await import('/skills/read-a-deck.js')         // some ship code as well
```

- **slides-and-speech**: a silent slide deck under a long recording, start to
  finish: read every state the deck reaches, time each to its line, hold the rest.
- **read-a-deck**: every state a deck reaches, in one decode pass, including
  the small text a whole-frame difference cannot see.
- **fix-a-moment**: a moment reported by timecode against a cut that already
  exists: read the live sequence, strip forward, split the hold in place.
- **verify-a-cut**: pictures first, then the counts after a reload.

They are also plain files under `/skills/`, for a client that would rather fetch them directly
better.
