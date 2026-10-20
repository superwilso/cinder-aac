# How the NW-A50 Bluetooth stack sends audio, and where AAC fits

Notes from reverse engineering for interoperability, in our own words. Function names are the
exported symbol names; nothing here is copied from Sony's or MediaTek's code. Firmware: stock
1.02 and Walkman One 3.02 (the Bluetooth libraries are byte-identical between them).

## The three layers

| layer | where it runs | what it does for a transmit link |
|---|---|---|
| `libBtMw.so` (Sony middleware) | hagodaemon `BtCommonService …` | holds the negotiated codec, starts the matching encoder, feeds it PCM from the transmitter socket |
| `libbluetooth.blueangel.so` (MediaTek) | same process | builds the endpoint list, runs the SBC/LDAC/aptX encoders, sends encoded frames to `mtkbt` |
| `mtkbt` (MediaTek) | its own process | AVDTP: advertises the endpoints, picks the codec, builds the RTP packets |

blueangel and mtkbt talk over datagram sockets in `/tmp` (`bt.int.adp`, `bt.a2dp.stream`).

## What already supports AAC

- **mtkbt** converts an AAC endpoint entry into AVDTP capabilities, and when choosing a codec
  as source tries, in order: LDAC, aptX HD, aptX, **AAC**, MP3, SBC. AAC is only considered while
  its AAC switch is on (message `0x222`, the same one the receiver's AAC setting sends).
- **mtkbt's packet path** is codec-agnostic: it copies the payload, adds the content-protection
  byte if SCMS-T is on, and adds the caller's step to the RTP timestamp after each packet.
- **libBtMw** reads an AAC link's sample rate correctly.
- **The player ships an AAC encoder**: `libFraunhoferAAC.so` exports `aacEncOpen` /
  `aacEncEncode`. Only its decoder is used (by the receiver).

## What is missing

1. **No AAC source endpoint.** blueangel's `btmtk_a2dp_sep_default_setting_initialized` fills four
   source entries: SBC, LDAC, aptX HD, aptX. mtkbt accepts at most four.
2. **libBtMw drops PCM for any codec but SBC, LDAC and aptX.** Its send-data request checks the
   codec and has no branch for AAC, and its encoder start does nothing for AAC either.
3. blueangel has no AAC encoder call.

## How cinder-aac fills the gaps

All hooks are functions blueangel calls through its own PLT (it is linked without `-Bsymbolic`,
with `BIND_NOW`), so an `LD_PRELOAD` in front of the service process stands in for them.

| hook | does |
|---|---|
| `btmtk_a2dp_sep_default_setting_initialized` | after blueangel fills the source list, replaces entry 2 (aptX HD) with AAC-LC, 44.1/48 kHz, stereo, up to 320 kbit/s |
| `btmtk_a2dp_send_aac_enable_req` / `btmtk_a2dp_send_activate_req` | keeps mtkbt's AAC switch on |
| the codec-config slot of `bt_av_callbacks` (exported) | when a *source* link comes up AAC, libBtMw is told it is SBC at the same rate, so it runs its SBC path |
| `btmtk_a2dp_stream_data_encoder_init` / `_deinit` | libBtMw's "SBC" encoder start/stop opens/closes an AAC encoder instead |
| `btmtk_a2dp_send_audio_stream_data` | libBtMw's PCM, 1024 frames at a time, encoded to LATM (StreamMuxConfig in every frame), paced to real time, handed to `btmtk_a2dp_send_audio_encoded_stream_data` with a timestamp step of 1024 |

Source vs receiver is told apart with libBtMw's exported `btmwAvBothIsSrc`.

## Trade-offs

- **aptX HD is gone while AAC is on.** Headphones with aptX HD fall back to aptX (they all have
  it), so only the HD upgrade is lost.
- **The receiver's AAC switch is held on** too while AAC transmit is on.
- **Sony's own status reports SBC** for an AAC link; `/tmp/cinder_aac_status` has the truth
  (`AAC <rate> <bitrate>`).
- **Priority** stays mtkbt's: LDAC > aptX > AAC > SBC. Headphones with LDAC or aptX still get
  those unless Cinder's codec choice switches them off.

## Files on the player

| path | what |
|---|---|
| `/system/vendor/unknown321/lib/preload/BtCommonService.so` | this library; Cinder's hagodaemon wrapper preloads any `preload/<Service>.so` into that service |
| `/data/cinder/bt_aac` | present = AAC on (read when the service starts) |
| `/tmp/cinder_aac.log` | what the hooks did |
| `/tmp/cinder_aac_status` | `AAC <rate> <bitrate>` while an AAC link is streaming |
