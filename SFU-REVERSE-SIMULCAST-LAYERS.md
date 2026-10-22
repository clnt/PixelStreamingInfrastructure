# SFU: simulcast layers in the order mediasoup expects (UE 5.8)

Investigation, change and testing by **Matt Clinton** (matt@mattc.dev). Questions and test results are welcome at
that address.

Branch `sfu-reverse-simulcast-layers-5.8`, one commit on top of Epic's `UE5.8` branch. It changes only
`SFU/sfu_server.js` (about 60 lines added) and documents the new setting in `SFU/README.md`. Nothing changes for the
signalling server, the frontend or Unreal.

It goes with a change to Unreal's Pixel Streaming 2 plugin for UE 5.8.3, published separately:
https://github.com/clnt/UnrealEngine/tree/ps2-fixes-5.8.3-sfu-simulcast (a private fork of Epic's UnrealEngine
repository, visible to GitHub accounts linked to Epic Games), file `PS2-SFU-SIMULCAST-5.8.3.md`. That change keeps
RTX in the offer Unreal sends the SFU, without which stock 5.8.3 simulcast never reaches its top layer. This SFU
change only takes effect for offers that include RTX (see "Only when the offer has RTX").

## The problem

Through the SFU with simulcast, a viewer receives **no video at all for ~2 s (up to ~12 s)** shortly after joining,
and again when a big scene change raises the bitrate. It doesn't drop to a smaller layer; the SFU stops forwarding to
that viewer, then resumes on the same layer. Measured on a 4K stream with 4K / 1440p / 1080p layers and three viewers
joining and leaving: 55 such stalls in 30 minutes, 22 of them 3 s or longer. Viewers on a non-simulcast stream through
the same SFU never stall.

## Why

- Unreal offers its simulcast encodings **full size first** (`simulcast0` = full size), and Epic's frontend relies on
  that: layer preference 0 ("High", which it selects on every join) means the full-size layer.
- mediasoup assumes encoding 0 is the **smallest**. In `SimulcastConsumer::IncreaseLayer` (mediasoup 3.15.5) a
  consumer with no layer walks the encodings from 0 upwards and takes the first one only if its whole current bitrate
  fits the consumer's bandwidth estimate. The preferred layer is the highest index the consumer may receive.
- So every viewer needs the full-size layer's entire bitrate before it receives anything, and a viewer on "High" is
  allowed only the full-size layer. Whenever its estimate dips below that layer's bitrate, mediasoup selects no layer
  (`layer changed! undefined` in the SFU log) and the viewer's video stops until the estimate recovers.

Unreal can't simply send the layers smallest first instead: the frontend would then select the smallest layer for
"High".

## The change

In `sfu_server.js`:
- when the streamer produces simulcast video, its encodings are handed to mediasoup **smallest first**
  (`simulcast2, simulcast1, simulcast0`);
- the spatial layer in a peer's `layerPreference` message is mapped to match, so 0 is still the full-size layer, 1
  the next, and so on. The frontend, Unreal and the signalling server see no difference. (A new consumer still
  starts with mediasoup's default preference, the highest layer, which is again the full-size one.)

So when a viewer's estimate dips, mediasoup falls back to a smaller layer for a moment instead of sending nothing, and
climbs back as the estimate recovers. Non-simulcast streams are untouched.

**Setting:** on by default. `--reverseSimulcastLayers=false` on the command line, or `reverseSimulcastLayers: false` in
`config.js`, restores Epic's behaviour. At start-up the SFU logs `reverseSimulcastLayers: true|false`, and for each
streamer it logs which order it used:
```
Simulcast layers given to mediasoup smallest first: simulcast2,simulcast1,simulcast0
Simulcast layers left in the streamer's order: the offer has no RTX
```

### Only when the offer has RTX

Stock UE 5.8.3 leaves RTX out of its offer. Its full-size layer then never becomes active, and its smallest layer
carries libwebrtc's bandwidth-probe padding, so mediasoup's score for that layer keeps dropping to 0 and back several
times a second. Put first in mediasoup's walk, that layer made viewers switch down to it and back, and the switches
stalled: 15 stalls in 10 minutes, against 4 in 30 minutes with Epic's order (where the walk starts at the middle,
first active, layer). So the SFU reorders only when the producer's codecs include `video/rtx`. With stock 5.8.3
Unreal it keeps Epic's order and behaves exactly like Epic's SFU; with Unreal that keeps RTX (5.8.3 with the plugin
change above, or 5.7.4) it reorders.

## Tests

Rig: RTX 3090, driver 610.88, Windows 11, UE 5.8.3 packaged Win64 Development, a 4K stream at 30 fps (H264), this SFU
and Epic's signalling server on the same PC. Three real Chrome viewers through the SFU joining and leaving every
20–40 s, a controller page sending a UI command every 20–30 s (large bitrate swings) and a viewer asking for another
layer every 30 s. Unreal with the RTX change and 4K / 1440p / 1080p layers unless stated.

Stall counts from an instrumented copy of the SFU with the same reordering (Epic's script doesn't log stalls; a stall
= the SFU forwarded nothing to a viewer):

| Run | Stalls (3 s or longer) | Viewers on 4K |
|---|---|---|
| Epic's order, 10 min | 21 (7), 51 s in all | 73% |
| Epic's order, 30 min | 55 (22) | 81% |
| Reordered, 10 min | 1 (0), 1 s | 75% |
| Reordered, 2 × 30 min | 0 and 2 (0), ~1 s each | 67–70% |
| Unreal's stock layers (4K / 1080p / 540p), Epic's order, 30 min | 36 (4) | 74% |
| Unreal's stock layers, reordered, 30 min | 0 | 76% |

This `sfu_server.js`, run as is:

| Run | Consumers with no layer (`layer changed! undefined`) | Viewer freezes (≥ 3 s) | Viewers received |
|---|---|---|---|
| Reordering on, 15 min | 2 in 31 joins | 0 | 4K 74%, 1440p 13%, 1080p 12% |
| `--reverseSimulcastLayers=false`, 10 min | 32 in 23 joins | 3 | 4K 63%, 1440p 19%, 1080p 18% |
| Stock UE 5.8.3 Unreal (no RTX), reordering on, 10 min | 0 in 21 joins | 0 | 1080p 100%, as with Epic's SFU |

Layer requests from the frontend still land on the right size (High → 4K, Medium → 1440p, Low → 1080p), in a median
0.6–0.8 s. No run had a GPU fault, an encoder error or memory growth.

[Longer runs with frequent viewer bandwidth changes are in progress; results will be added here.]

## Applying it elsewhere

The change is self-contained in `SFU/sfu_server.js`. To apply it to another copy of Epic's UE5.8 SFU:
```
git diff a5b526b2 sfu-reverse-simulcast-layers-5.8 -- SFU/sfu_server.js > sfu_server-reverse-layers.patch
git apply sfu_server-reverse-layers.patch     # in the other checkout's root
```
