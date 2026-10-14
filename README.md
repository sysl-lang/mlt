# mlt

[MLT](https://www.mltframework.org/), the multimedia framework behind Shotcut and Kdenlive, bound
for sysl: profiles, producers, filters, transitions, playlists, tractors, consumers, frames with their
pixels and samples, and events — including the ones a consumer fires from its render thread.

```hocon
dependencies {
  mlt { git = "github.com/sysl-lang/mlt", version = "0.1.0" }
}
```

```sysl
import sh.sysl.mlt.*
```

MLT 7 has to be installed with its plugins (`brew install mlt`; Debian's `libmlt-dev`, `libmlt-data`
and `melt`). The build asks pkg-config for `mlt-framework-7`. MLT is LGPL-2.1; this package is ISC.

## An example

Two coloured clips, one second each, rendered to an MP4, with a listener counting frames as the
consumer shows them. This is `example_tests.sysl`, compiled and run by `sysl test .`:

```sysl
render_title(file: string) -> Result[int, Error]
    val prof = profile("atsc_720p_25")?
    val show = playlist(prof)?

    show.append_io(producer(prof, "color:#202020")?, 0, 24)?
    show.append_io(producer(prof, "color:#c04000")?, 0, 24)?

    val out = consumer(prof, "avformat", file)?
    val progress: &sync Progress = Progress(Atomic(0))

    out.set("vcodec", "mpeg4")?
    out.set_int("an", 1)?
    out.on("consumer-frame-show", (e) -> progress.frames.add(1))
    out.connect(show)?
    out.run()?

    Ok(progress.frames.load())
```

## How it is shaped

- **Every MLT object is a `&T` that owns one MLT reference**, and its `Drop` calls the matching
  `mlt_*_close` exactly once. MLT counts the rest: a playlist that appends a producer, a tractor that
  takes a track, a consumer that connects to a source each take their own, so dropping the sysl value
  does not stop it playing there. Where MLT *lends* an object (`Playlist.clip`, `Tractor.track`,
  `Event.frame`) the binding takes a reference of its own before wrapping it.
- **Every object is a property bag** through the `Object` trait: `get`/`set`, `get_int`, `get_int64`,
  `get_double`, `get_position`, `get_color`, `get_time`/`frames_to_time`/`time_to_frames` in any of
  the four `TimeFormat`s, `ref_count`, and `on`.
- **`Source`** is anything a consumer can take frames from: `Producer`, `Playlist`, `Tractor`.
  `frame()` and `frame_at(n)` pull one directly.
- **Every service holds the `&Profile` it was made with**, because MLT keeps only a pointer to it.
- **`init()` loads the plugin repository once**, from `MLT_REPOSITORY` or the library's own path; every
  constructor calls it. `init(deny)` names modules not to load, through MLT's `MLT_REPOSITORY_DENY`.
- **Errors** are `UnknownService(kind, name)`, `MissingFile(path)`, `InvalidProfile(name)`,
  `NoRepository(dir)`, `Failed(call, code)` and `NoMedia(what)`. MLT itself answers only a null
  pointer or a non-zero integer, so each variant is what the binding knew about the call it made.

### A program with no window should deny `sdl2`

Loading MLT's `sdl2` module loads Homebrew's sdl2-compat, which `dlopen`s `libSDL3.dylib` beside the
executable and in the system's library directories, not in `/opt/homebrew/lib`. When that fails it
puts a modal **"Failed loading SDL3 library"** dialog on the screen and waits for someone to dismiss it.
The tests call `init(["sdl2", "qt6", "rtaudio", "decklink", "opencv"])` in a `@setup` that runs first
in every test process, and `denied_modules_are_absent` proves none of those services is reachable.
`qt6` is a GUI toolkit and `rtaudio` and `decklink` open hardware; `opencv` brings in TBB, whose
exit-time destructor calls into unmapped code in a process under AddressSanitizer.

### Events from the render thread

A started consumer renders on a thread of its own and fires `consumer-frame-show` there, so `on` takes
a `&sync Fn(Event) -> unit`. The compiler checks what the closure captures where it is written: a
`&sync` struct of `Atomic`s, a scalar or a `*T` is accepted; a plain `&T` or a `string` is refused.
The listener is boxed on the object, MLT's `listener_data` points at the box, and one `@export`ed
trampoline (`mlt_listener` takes `mlt_event_data` by value, so it needs a C-convention address) calls
the closure. Dropping the object disconnects its listeners before closing it, so an MLT object kept
alive elsewhere cannot call into a freed box.

### Pixels and samples

`Frame.image(format)` answers an `Image` of `width`, `height` and `pixels: []const u8`;
`Frame.audio(format, frequency, channels)` an `Audio` whose `data` is the samples. Both hold the
`Frame`, so the bytes stay valid for as long as the `Image` or `Audio` is held. **A slice copied out of
one is valid only while the `Image`, `Audio` or `Frame` is still held**: a view of memory C owns
carries no reference to anything (see "Gaps" below).

## The survey: what is bound, and what is deliberately absent

The public C API (`framework/mlt.h`) declares about 620 functions in 31 headers; the library exports
573 `mlt_*` symbols. Classified:

| area | functions | bucket | bound |
|---|---:|---|---|
| factory, repository, environment | 27 | expressible | init, the four service lists, directory, environment |
| profile | 15 | expressible (`mlt_profile_s` is a published struct, size asserted) | init, load file/string, fps/sar/dar, valid, close |
| properties | 72 | expressible | string/int/int64/double/position/colour/time, exists, clear, count, refs |
| property (single values), animation, tokeniser, parser | 76 | expressible | not bound — internal to properties and to service authors |
| events | 23 | **shim-able, no shim needed**: `mlt_event_data` is an untagged union, carried as its one word and read through MLT's own `mlt_event_data_to_*` | listen, disconnect, to int/string/frame |
| service | 28 | expressible | attach, detach, filter count, get frame, profile |
| producer | 33 | expressible | seek, position, in/out, length, playtime, speed, fps, cut, close |
| filter, transition | 32 | expressible | factory, in/out, tracks, close |
| playlist | 44 | expressible | new, append, append_io, blank, remove, clip, clip start/length, blank test, mix |
| tractor, multitrack, field | 36 | expressible | new, set/get track, track count, plant filter/transition |
| consumer | 15 | expressible | factory, connect, start, stop, is_stopped, position, close; `run`/`wait` |
| frame | 37 | expressible | properties, position, get_image, get_audio, close |
| image, audio | 47 | expressible | format sizes, samples per frame |
| `mlt_*_init(self, child)`, `*_new` for writing a service *in* sysl, `get_image` callbacks, frame stacks, `mlt_frame_set_image` with a destructor | ~40 | needs the language to move for the image/audio adoption half (see Gaps); the service-authoring half is expressible but out of scope | not bound |
| cache, slices, pool, deque, log (variadic `mlt_log`), luma map, chain/link | ~60 | expressible (`mlt_log` is variadic and callable) | not bound: internal machinery or service authoring |
| `mlt_properties_dump`/`debug` (take a `FILE *`) | 2 | shim-able | not bound |
| `mlt_properties_serialise_yaml`, `mlt_profile_lumas_dir` (answer `malloc`'d strings) | 2 | expressible with `free` | not bound yet; `free` is declared for them |

Totals: about **575 expressible**, **3 shim-able** (the event-data union, which needed no shim, and two
`FILE *` dumps), and **the image/audio adoption calls** in the third bucket.

**Deliberately absent: the `sdl2`, `qt6`, `rtaudio`, `decklink` and `opencv` modules in the tests**,
for the reasons above; a program that has a window and SDL3 where sdl2-compat looks can use them.

## Gaps found in sysl

- **A slice over memory C owns cannot keep the C owner alive.** A view's owner word is any counted
  object, but nothing lets a program *name* one: `p[0..<n]` on a `*T` owns nothing. So
  `Frame.image().pixels`, copied out, dangles once the frame goes — the shape an MLT frame, an LMDB read
  and a decoder's output buffer all have. The binding returns the slice beside the `&Frame` that keeps
  it alive instead of being able to make the slice itself carry it.

## Tests

`sysl test .` — 23 tests. Each compares against something that is not MLT's word for itself: a colour
producer's every pixel against the bytes its hex spelling names (RGBA and RGB); an MP4 rendered through
`avformat` against `ffprobe` (codec, width, height, frame count, duration); the `xml` consumer's
document for a playlist and a tractor against the document written out by hand; a listener counting
`consumer-frame-show` from the render thread (and proving it is not the caller's thread) against the
frame count; timecodes against arithmetic; and the error paths — an unknown producer, filter,
transition and consumer, a missing file both through the loader and through `avformat`, an unknown
profile and an invalid profile description — each against the exact `Error`.

Under AddressSanitizer: `SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .`. This is a
`pkg_config` package, so **only the sysl half is instrumented** — MLT and its plugins are Homebrew's
uninstrumented builds.

## License

ISC. See `LICENSE`.
