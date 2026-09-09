# bilbycast-decklink-rs

Safe Rust SDI **capture and playout** for **Blackmagic DeckLink** cards,
talking to the Blackmagic DeckLink SDK directly. Backs
[bilbycast-edge](https://github.com/Bilbycast/bilbycast-edge)'s `sdi-decklink`
feature (off by default), targeting upstream issue
[#19](https://github.com/Bilbycast/bilbycast-edge/issues/19).

```
bilbycast-decklink-rs/
├── libdecklink-sys/   C++ shim exposing a C ABI over the SDK, + bindgen FFI
│   └── shim/          decklink_shim.{h,cpp}, decklink_shim_playout.cpp
└── decklink-rs/       safe wrapper: DecklinkCapture, DecklinkPlayout,
                       device_status, enumerate_devices, api_available,
                       VANC ancillary capture + playout
```

## Why the SDK and not FFmpeg's `decklink` avdevice

FFmpeg's avdevice works, but it hides `bmdFrameHasNoInputSource`: on signal loss
it silently substitutes colour bars, so **a pulled cable is indistinguishable
from a healthy feed**. Going to the SDK directly also avoids a lot of incidental
pain:

* no `--enable-decklink --enable-nonfree` FFmpeg build — the edge binary stays
  redistributable;
* no FFmpeg >= 8 requirement;
* no duplicate `libav*` symbols in a binary that already links FFmpeg;
* device enumeration that doesn't wedge the card.

## Build

Only the SDK **headers** are needed at build time. `DeckLinkAPIDispatch.cpp`
`dlopen`s `libDeckLinkAPI.so` at runtime.

```bash
# Blackmagic "Desktop Video SDK" -> Linux/include
export DECKLINK_SDK_DIR=/path/to/Blackmagic_DeckLink_SDK/Linux/include
cargo build

# On a host with a card + Desktop Video installed:
cargo run -p decklink-rs --example list_devices
cargo run -p decklink-rs --example capture_probe -- "DeckLink Quad (1)" auto

# Paired VANC loopback (playout on one sub-device, capture on the looped port):
cargo run -p decklink-rs --example playout_ancillary -- "DeckLink Quad (1)"
cargo run -p decklink-rs --example capture_ancillary -- "DeckLink Quad (2)"
```

Runtime requires Blackmagic **Desktop Video** (kernel driver + `libDeckLinkAPI.so`).

See [CLAUDE.md](CLAUDE.md) for design notes, the known-good bilbycast-edge
config, and the upstream bugs found during bring-up.

## Status

* `enumerate_devices` and `DecklinkCapture` — implemented, verified on a
  DeckLink Quad 2 against a live 1080i50 source.
* `device_status` — read-only `IDeckLinkStatus` snapshot (signal lock, genlock /
  reference lock, detected raster + colorimetry + field dominance, SDI link
  configuration, PCIe link speed/width, busy). Opens nothing, so it works on
  every port of a card while flows are running. Every field is `Option`: the
  card answers per-field and "unknown" is never rendered as "no".
* `DecklinkPlayout` — scheduled **video + audio playout** against the card's
  clock (3-frame pre-roll, completion-callback-paced in-flight window,
  late/dropped counters). Both essences are scheduled on one 90 kHz timeline
  the caller owns — pass each as `essence_pts - first_video_pts` and the card
  lip-syncs them in hardware. Audio is 48 kHz 32-bit interleaved. Verified by
  physical BNC loopback: bars/video out one sub-device, captured on the looped
  port — correct colours, live motion, continuous audio, `late=0 dropped=0`.
* **Generic VANC ancillary** — `CapturedVideo.ancillary` on capture,
  `DecklinkPlayout::write_video_with_ancillary` on playout. Protocol-agnostic
  `did` / `sdid` / `line_number` / `data`; the shim filters to
  `bmdAncillaryDataSpaceVANC` (HANC is not surfaced) and copies the bytes out
  inside the capture callback. Nothing here knows what SCTE-104 is — payload
  semantics belong to the caller. Hardware-loopback verified at 1080i50, exact
  byte-for-byte SCTE-104 recovered; see CLAUDE.md for the two undocumented SDK
  requirements the write side has to satisfy.
* `api_available` — whether the API itself is reachable (Desktop Video
  installed, `libDeckLinkAPI.so` dlopened): it creates and immediately releases
  an iterator, opening no card. `enumerate_devices` returns empty for *both* a
  missing driver and a driver with no card fitted; this is what separates them.

On 8-port Quad cards the physical→software connector mapping interleaves and
sub-device pairs share connectors, so `enumerate_devices` reports the BNC as
`physical_port` (`None` on any model whose layout is not verified) — see
CLAUDE.md before touching a rack.

## License

MPL-2.0. The DeckLink SDK itself is Blackmagic's and is not redistributed here —
only our own shim sources.
