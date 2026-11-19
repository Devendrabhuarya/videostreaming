# Adaptive UDP Video Streaming System in C: Architecture

A sender streams H.264 video over UDP to a receiver. The receiver hides loss, reordering and jitter with a jitter buffer and reports network conditions back to the sender. The sender uses AIMD congestion control to pick a sending rate and ABR to pick the video quality level that fits that rate.

---

## 1. Goals and scope

| Goal | How it is met |
|---|---|
| Stream H.264 over raw UDP | POSIX sockets, custom packet format |
| Survive loss, reordering, jitter | Sequence numbers, fragment reassembly, receiver jitter buffer |
| Be network-friendly | Rate-based AIMD congestion control driven by receiver feedback |
| Adapt quality | ABR over a pre-encoded ladder of keyframe-aligned variants |
| Be testable | Network emulator with configurable loss, delay, jitter, bandwidth |
| Be observable | CSV metrics on both sides, plot script |

**Non-goals (v1):** encryption, multi-client fan-out, retransmission/FEC (listed under extensions), audio.

---

## 2. System overview

```
 SENDER                                NETEM (optional)                   RECEIVER
+--------------------------+        +--------------------+        +---------------------------+
| media_source             |        |  loss / delay /    |        | UDP recv                  |
|  (reads variant .h264)   |        |  jitter / reorder  |        |   |                       |
|        |                 |  data  |  / bandwidth cap   |  data  |   v                       |
|   packetizer             |------->|  (forward path)    |------->| reassembler               |
|        |                 |        |                    |        |   |                       |
|   pacer (token bucket)   |        |                    |        |   v                       |
|        |                 |        |                    |        | jitter buffer             |
|   UDP send               |        |                    |        |   |                       |
|                          |        |                    |        |   v                       |
| rate_control (AIMD) <----+-----+  |                    |  +-----+ playout --> stdout --> ffplay
| abr (level selector)     |     |  |  (reverse path)    |  |     | feedback_tx (every 100ms) |
+--------------------------+     +--+<-------------------+<-+     +---------------------------+
                                          feedback
```

**Data flow (forward):** access unit (one encoded frame) -> fragments -> paced UDP packets -> network -> reassembly -> jitter buffer -> in-order playout -> `ffplay`.

**Control flow (reverse):** receiver measures loss, jitter, receive rate, buffer depth -> feedback packet -> sender AIMD updates target rate -> ABR picks quality level for the next GOP.

---

## 3. Tech stack and dependencies

- **Language:** C11, GCC or Clang, Linux (POSIX sockets, `timerfd`, `poll`, pthreads)
- **FFmpeg libraries:** `libavformat`, `libavcodec`, `libavutil` (sender demuxing). Install with `sudo apt install ffmpeg libavformat-dev libavcodec-dev libavutil-dev pkg-config`
- **Tools:** `ffmpeg` (encoding variants), `ffplay` (playback), `tc` (optional kernel-level emulation), Python + matplotlib (plots)
- **Build:** Make (CMake also fine)

```make
CFLAGS  = -std=c11 -O2 -Wall -Wextra -Wpedantic -pthread -Iinclude $(shell pkg-config --cflags libavformat libavcodec libavutil)
LDFLAGS = -pthread -lm $(shell pkg-config --libs libavformat libavcodec libavutil)
```

---

## 4. Repository layout

```
udp-abr-stream/
├── Makefile
├── ARCHITECTURE.md
├── README.md
├── include/
│   ├── protocol.h        # packet formats, constants, (de)serialization
│   ├── packetizer.h      # frame -> fragments
│   ├── pacer.h           # token-bucket pacer
│   ├── rate_control.h    # AIMD
│   ├── abr.h             # quality selection
│   ├── jitter_buffer.h   # frame slots, playout scheduling
│   ├── netem.h           # emulator internals
│   ├── metrics.h         # CSV logging
│   └── util.h            # time, logging, seq arithmetic
├── src/
│   ├── common/   protocol.c  util.c  metrics.c
│   ├── sender/   main.c  media_source.c  packetizer.c  pacer.c  rate_control.c  abr.c
│   ├── receiver/ main.c  reassembler.c  jitter_buffer.c  playout.c  feedback_tx.c
│   └── netem/    main.c
├── scripts/
│   ├── encode_variants.sh
│   ├── run_demo.sh
│   └── plot_metrics.py
├── tests/        test_protocol.c  test_packetizer.c  test_jitter_buffer.c  test_aimd.c
├── media/        my_video.mp4   variants/
└── logs/
```

Three binaries: `sender`, `receiver`, `netem`. Shared code in `src/common` is built as a static library `libcommon.a`.

---

## 5. Media preparation (ABR ladder)

Encode the source into several variants **offline**. The sender never encodes; it reads already-encoded access units and switches between files. This keeps the sender simple and fast.

**Hard requirements for every variant:**

1. Same frame rate and same frame count.
2. **Keyframe-aligned:** an IDR frame every `GOP` frames (for example 30 frames = 1 s at 30 fps) at identical positions in every variant. Quality switches happen only at these points.
3. SPS/PPS repeated before every IDR so a decoder can start or switch at any keyframe.
4. No B-frames (`-bf 0`) so decode order equals display order and packet order equals frame order.
5. Annex-B byte-stream format.

| Level | Resolution | Target bitrate |
|---|---|---|
| 0 | 240p | 300 kbps |
| 1 | 360p | 600 kbps |
| 2 | 480p | 1200 kbps |
| 3 | 720p | 2500 kbps |

`scripts/encode_variants.sh`:

```bash
#!/usr/bin/env bash
set -e
SRC=media/my_video.mp4; OUT=media/variants; mkdir -p "$OUT"
FPS=30; GOP=30
encode() { # name height bitrate
  ffmpeg -y -i "$SRC" -an -vf "scale=-2:$2" -r $FPS \
    -c:v libx264 -preset veryfast -profile:v main \
    -b:v "$3" -maxrate "$3" -bufsize "$(( ${3%k} * 2 ))k" \
    -g $GOP -keyint_min $GOP -sc_threshold 0 -bf 0 \
    -x264-params "repeat-headers=1:aud=1" \
    -bsf:v h264_mp4toannexb -f h264 "$OUT/$1.h264"
}
encode q0_240p 240 300k
encode q1_360p 360 600k
encode q2_480p 480 1200k
encode q3_720p 720 2500k
```

> **Resolution-switch caution:** some players re-initialise or glitch when resolution changes mid-stream. If `ffplay` misbehaves, encode all levels at the same resolution with different bitrates, which keeps ABR logic identical and avoids the issue.

**Loading variants (sender):** open each file with `avformat_open_input` (raw `h264` demuxer), read every packet with `av_read_frame` once at startup, and build an in-memory index per level: `frames[level][idx] = { data, size, is_key }`. For a short clip this fits in RAM and makes switching trivial: to switch level at frame `idx`, just read `frames[new_level][idx]`. Assign `pts_ms = idx * 1000 / FPS` yourself; do not rely on stream timestamps.

---

## 6. Wire protocol

All multi-byte fields are **big-endian (network byte order)**. Serialize field by field with `htons/htonl`; never `memcpy` a struct onto the wire.

### 6.1 Data packet (sender -> receiver)

Header = 30 bytes, payload ≤ 1200 bytes, so the datagram is ≤ 1230 bytes and never IP-fragments on a 1500 MTU path.

| Field | Size | Meaning |
|---|---|---|
| `magic` | 2 | `0x5653` ("VS"), reject anything else |
| `version` | 1 | `1` |
| `type` | 1 | `1 = DATA` |
| `flags` | 1 | bit0 `KEYFRAME`, bit1 `LAST_FRAG`, bit2 `SWITCH_POINT`, bit3 `EOS` |
| `quality` | 1 | ABR level index of this frame |
| `frag_idx` | 2 | fragment index within the frame |
| `frag_cnt` | 2 | total fragments in the frame |
| `seq` | 4 | per-packet counter (monotonic across the whole stream) |
| `frame_id` | 4 | frame index (= `idx` above) |
| `pts_ms` | 4 | media timestamp |
| `send_ts_ms` | 4 | sender clock at send time (for jitter/RTT) |
| `payload_len` | 2 | bytes of payload |
| `reserved` | 2 | zero (future checksum) |

### 6.2 Feedback packet (receiver -> sender), every ~100 ms

| Field | Size | Meaning |
|---|---|---|
| `magic/version/type` | 4 | `type = 2 = FEEDBACK` |
| `flags` | 1 | bit0 `WANT_KEYFRAME` |
| `cur_quality` | 1 | level the receiver is currently playing |
| `fb_seq` | 2 | feedback counter |
| `highest_seq` | 4 | highest data `seq` seen |
| `expected_interval` | 4 | packets expected since last feedback |
| `received_interval` | 4 | packets received since last feedback |
| `loss_permille` | 2 | loss fraction ×1000 over the interval |
| `jitter_us` | 4 | smoothed interarrival jitter (RFC 3550) |
| `recv_rate_kbps` | 4 | measured goodput over the interval |
| `buffer_ms` | 2 | jitter buffer depth (media time queued) |
| `echo_send_ts_ms` | 4 | `send_ts_ms` of the latest data packet received |
| `echo_hold_ms` | 4 | time the receiver held that timestamp before sending feedback |

RTT = `now - echo_send_ts_ms - echo_hold_ms`.

### 6.3 Control packets

`HELLO` (receiver -> sender: "start streaming"), `STREAM_INFO` (sender -> receiver: fps, GOP, number of levels, bitrates), `BYE`. Resend control packets up to 3 times with a 500 ms timeout; data packets are never retransmitted in v1.

### 6.4 Sequence-number arithmetic

`seq` and `frame_id` are 32-bit and will wrap in long runs. Always compare with serial-number arithmetic:

```c
static inline int32_t seq_diff(uint32_t a, uint32_t b) { return (int32_t)(a - b); }
/* a is newer than b  <=>  seq_diff(a, b) > 0 */
```

---

## 7. Sender design

Single-threaded `poll()` event loop; no locks needed.

**File descriptors polled:** UDP socket (feedback in), `timerfd` frame clock (1/fps), `timerfd` pacer tick (1-2 ms).

```
state: IDLE --HELLO--> STREAMING --last frame--> DRAINING --BYE/timeout--> DONE
```

### 7.1 Frame tick

```
on frame_timer:
    idx = next_frame_idx++
    if idx % GOP == 0:
        level = abr_commit_switch()          // only switch on a keyframe boundary
    au   = media_source_get(level, idx)      // Annex-B access unit
    pkts = packetize(au, frame_id=idx, level, pts_ms)
    pacer_enqueue(pkts)
```

### 7.2 Packetizer / fragmentation

- Split `au` into `ceil(size / MAX_PAYLOAD)` fragments; each gets `frag_idx`, `frag_cnt`, the frame's `quality`, `pts_ms`, and a fresh `seq`.
- Set `KEYFRAME` on all fragments of an IDR frame, `LAST_FRAG` on the final one.
- Fragmenting at the frame level means the receiver can detect "frame complete" with a bitmap and nothing else.

### 7.3 Pacer (token bucket)

Do not blast a whole I-frame (often 20-100 KB) in one burst: it overflows router queues and looks like loss.

```
on pacer_tick(dt):
    tokens = min(tokens + rate_Bps * dt, BURST_BYTES)       // BURST ≈ 2-4 packets
    while queue not empty and tokens >= head.size:
        sendto(head); tokens -= head.size; pop
```

**Backlog protection:** if queued data exceeds `MAX_QUEUE_MS` (say 300 ms of media at the current rate), drop the oldest whole non-key frames from the queue and tell ABR to step down. Never drop part of a frame.

### 7.4 Feedback handling

```
on feedback_packet:
    rate_control_on_feedback(fb)       // AIMD (section 9)
    abr_on_rate_update(rate, fb)       // ABR (section 10)
    if fb.WANT_KEYFRAME: abr_request_resync()   // jump to next GOP boundary
```

---

## 8. Receiver design

Two threads sharing the jitter buffer behind one mutex and one condition variable.

- **Network thread:** `poll()` on the UDP socket with a 100 ms timeout. It parses and validates packets, updates statistics, inserts fragments, and sends feedback when the timer expires.
- **Playout thread:** waits for each frame's playout deadline, pulls the frame, writes it to stdout.

### 8.1 Reassembler / jitter buffer

A ring of frame slots indexed by `frame_id % N_SLOTS` (N ≈ 128):

```c
typedef struct {
    uint32_t frame_id;
    uint32_t pts_ms;
    uint16_t frag_cnt, frags_have;
    uint64_t frag_bitmap[ (MAX_FRAGS + 63) / 64 ];
    uint8_t  is_key, quality, state;   // EMPTY, FILLING, COMPLETE
    uint32_t size;
    int64_t  first_arrival_us;
    uint8_t  data[MAX_FRAME_BYTES];    // pre-allocated, no malloc on hot path
} frame_slot_t;
```

On each data packet:

1. Validate header (magic, version, `frag_idx < frag_cnt`, `payload_len` ≤ remaining).
2. Drop if the frame is already played or too old (`seq_diff(frame_id, next_play_id) < 0`).
3. Drop duplicates (bitmap bit already set).
4. `memcpy` the payload to `data + frag_idx * MAX_PAYLOAD`; set the bit; if `frags_have == frag_cnt`, mark `COMPLETE`.

Because slots are indexed by `frame_id` and fragments by `frag_idx`, **reordering is handled for free**.

### 8.2 Loss and jitter statistics

Track per stream: `base_seq`, `max_seq`, total `received`; per feedback interval: `expected_interval`, `received_interval`.

```
loss_fraction = (expected_interval - received_interval) / expected_interval
```

Interarrival jitter (RFC 3550), updated on every data packet:

```
D = (arrival_i - arrival_{i-1}) - (send_ts_i - send_ts_{i-1})
J = J + (|D| - J) / 16
```

This needs no clock synchronization because only differences are used.

### 8.3 Playout scheduling

On the first frame: `playout_base = local_now + initial_delay` (e.g. 200 ms) and `first_pts = pts_ms`. Then:

```
playout_time(frame) = playout_base + (pts_ms - first_pts)
```

**Adaptive delay:** target delay = `clamp(MIN_DELAY + 3 * J, MIN_DELAY, MAX_DELAY)`. Apply changes only at keyframe boundaries (shift `playout_base`) so viewers do not see speed-ups or slow-downs mid-GOP.

**Playout loop:**

```
for next_id = first_id ...:
    wait until playout_time(next_id)  (+ GRACE_MS)
    slot = slots[next_id % N]
    if slot.COMPLETE:            write(stdout, slot.data, slot.size)
    else:                        handle_missing(next_id)       // see below
    release slot; next_id++
```

**Missing / incomplete frame policy** (configurable):

- `DROP_TO_KEYFRAME` (default, clean picture): skip all following P-frames until the next IDR, set `WANT_KEYFRAME` in feedback, and the display holds the last good frame.
- `CONCEAL`: write the partial frame anyway and let FFmpeg's decoder conceal errors (can show artifacts, but no freeze).

With a 1-second GOP, a lost reference frame can freeze video for up to a second. A 0.5-second GOP trades some bitrate for faster recovery.

### 8.4 Output to ffplay

The receiver writes the Annex-B bitstream to stdout, and **all logs go to stderr**:

```bash
./build/receiver --listen 5000 --feedback-to 127.0.0.1:6001 --out - \
 | ffplay -f h264 -framerate 30 -fflags nobuffer -flags low_delay \
          -probesize 32 -analyzeduration 0 -i -
```

---

## 9. Congestion control (AIMD)

Rate-based: the sender maintains a target sending rate `R` (bits/s).

```
on_feedback(fb):
    rtt  = now - fb.echo_send_ts - fb.echo_hold
    loss = fb.loss_permille / 1000

    if loss > LOSS_HI                          // e.g. 0.02
       or fb.buffer_ms < BUF_LOW:              // receiver is starving
        if now - last_decrease > max(rtt, MIN_DECREASE_GAP):
            R = max(R_MIN, R * BETA)           // multiplicative decrease, BETA = 0.7
            last_decrease = now
    elif loss < LOSS_LO:                       // e.g. 0.005
        R = min(R_MAX, R + ALPHA)              // additive increase, ALPHA ≈ 50 kbps per feedback
    // else: hold

    R = min(R, 1.5 * fb.recv_rate)             // never run far above what actually arrived
    pacer_set_rate(R)
```

Notes:

- **Rate-limit decreases to once per RTT** so a burst of loss reports does not collapse the rate.
- Start at the lowest level's bitrate and ramp up (slow-start-like: use a larger `ALPHA` until the first loss).
- If no feedback arrives for `3 * FEEDBACK_INTERVAL`, halve `R` each timeout (feedback blackout).

---

## 10. Adaptive bitrate (ABR)

ABR converts the congestion-control rate `R` into a quality level.

```
abr_on_rate_update(R, fb):
    safe = SAFETY * R                          // SAFETY = 0.8: headroom for headers + variance
    target = highest level L with bitrate[L] <= safe

    if target < cur_level:                     // step down: immediately
        pending_level = target
    elif target > cur_level:                   // step up: cautiously
        if sustained(target OK for UP_HOLD_S seconds)        // e.g. 5 s
           and fb.buffer_ms >= BUF_OK
           and loss low:
            pending_level = cur_level + 1      // one step at a time
```

`abr_commit_switch()` runs at the next GOP boundary and applies `pending_level` (section 7.1). Set `SWITCH_POINT` on the first frame after a switch. Add a minimum dwell time (e.g. 3 s) between switches to avoid oscillation.

---

## 11. Network emulator (`netem` binary)

A UDP proxy placed between sender and receiver so you can test reproducibly on one machine.

```
sender --> [fwd port] --> impairments --> receiver
sender <-- [rev port] <-- (optional impairments) <-- receiver
```

Pipeline per forward packet:

1. **Loss:** Bernoulli with probability `p`, or Gilbert-Elliott (two-state good/bad Markov chain: `p_gb`, `p_bg`, `loss_in_bad`) for bursty loss.
2. **Bandwidth cap:** model a bottleneck link. `depart = max(now, link_free_at) + size*8 / bw; link_free_at = depart`. Tail-drop when queued bytes exceed `queue_limit`. This is what makes AIMD actually see congestion.
3. **Delay + jitter:** `release = depart + base_delay + jitter_sample` (uniform or normal).
4. **Reordering:** if `--reorder` is on, allow release times to be non-monotonic; otherwise clamp `release = max(release, last_release)`.
5. Insert into a **min-heap keyed by `release`**; a `timerfd` fires for the earliest entry and forwards the packet.

**Dynamic scenarios:** `--schedule schedule.csv` with rows `t_sec,bw_kbps,loss_pct,delay_ms,jitter_ms` so you can script "drop to 800 kbps at t=20 s, restore at t=40 s".

**Alternative:** Linux kernel emulation on loopback:

```bash
sudo tc qdisc add dev lo root netem delay 40ms 10ms loss 2% rate 2mbit
sudo tc qdisc del dev lo root       # clean up
```

Use `tc` as a cross-check on your emulator.

---

## 12. Concurrency and memory model

| Process | Threads | Shared state | Sync |
|---|---|---|---|
| sender | 1 (event loop) | none | none |
| receiver | network + playout | jitter buffer, stats | 1 mutex + 1 condvar (`pthread_cond_timedwait` with `CLOCK_MONOTONIC`) |
| netem | 1 (event loop) | heap | none |

- Use `CLOCK_MONOTONIC` everywhere; never wall-clock time.
- **No `malloc` on the packet hot path:** pre-allocate frame slots, packet pools, and the pacer queue at startup.
- Set `SO_RCVBUF` / `SO_SNDBUF` to ~1-4 MB so bursts do not overflow the socket buffer.
- Keep the mutex held briefly: copy the frame out, release, then do the `write()` to stdout.

---

## 13. Configuration

Command-line flags with a config-file fallback.

| Parameter | Default | Component |
|---|---|---|
| `FPS` / `GOP` | 30 / 30 | sender |
| `MAX_PAYLOAD` | 1200 B | both |
| `FEEDBACK_INTERVAL` | 100 ms | receiver |
| `R_MIN` / `R_MAX` | 250 kbps / 4 Mbps | sender |
| `ALPHA` / `BETA` | 50 kbps / 0.7 | AIMD |
| `LOSS_HI` / `LOSS_LO` | 2% / 0.5% | AIMD |
| `SAFETY` | 0.8 | ABR |
| `UP_HOLD_S` / `MIN_DWELL_S` | 5 s / 3 s | ABR |
| `INITIAL_DELAY` / `MIN_DELAY` / `MAX_DELAY` | 200 / 100 / 800 ms | jitter buffer |
| `GRACE_MS` | 20 ms | jitter buffer |
| `MISSING_POLICY` | `DROP_TO_KEYFRAME` | jitter buffer |
| `MAX_QUEUE_MS` | 300 ms | pacer |

---

## 14. Metrics and logging

Each binary writes a CSV to `logs/` once per second (and on every event of interest). Human-readable logs go to **stderr**.

- **Sender:** `t, target_rate_kbps, sent_kbps, level, rtt_ms, loss_permille, pacer_queue_ms, frames_dropped`
- **Receiver:** `t, recv_kbps, loss_pct, jitter_ms, buffer_ms, level, frames_played, frames_missed, stalls, late_packets, reordered_packets`
- **Netem:** `t, in_pkts, dropped_loss, dropped_queue, forwarded`

`scripts/plot_metrics.py` overlays target rate vs. the ladder bitrate vs. measured goodput, with loss and buffer depth below, which is the headline figure that shows AIMD + ABR working.

---

## 15. Testing and evaluation

**Unit tests** (plain C `assert`s, `make test`):

- `protocol`: round-trip encode/decode of every packet type, reject bad magic/length, endianness.
- `packetizer`: sizes 0, 1, `MAX_PAYLOAD`, `MAX_PAYLOAD+1`, 100 KB reassemble to identical bytes.
- `jitter_buffer`: in-order, reversed, duplicated, missing-fragment, late-frame, wrap-around of `seq`/`frame_id`.
- `aimd`: synthetic feedback sequences produce expected increase/decrease and once-per-RTT limit.

**Scenario tests** (via `scripts/run_demo.sh` and the emulator):

| Scenario | Expected behaviour |
|---|---|
| Clean link, 5 Mbps | Climbs to level 3, stays there |
| 2% random loss | Rate settles, mostly level 1-2, no stalls |
| Bandwidth step 5 Mbps -> 800 kbps at t=20 s | Down-switch within ~1-2 s, no long freeze |
| Bandwidth restored | Gradual one-level-at-a-time up-switch after hold time |
| 50 ms ± 30 ms jitter + reordering | No visible glitches; buffer delay adapts |
| Burst loss (Gilbert-Elliott) | Brief freeze at most until next keyframe |
| Feedback path blackout 3 s | Sender backs off, recovers when feedback resumes |

**Quality metrics:** stall count/duration, frames played vs. missed, average quality level, number of switches, end-to-end latency.

**Debugging aids:** run with `-fsanitize=address,undefined` in debug builds; capture with Wireshark on loopback (`udp.port == 6000`) and verify headers.

---

## 16. Build order (milestones)

1. **Skeleton:** Makefile, logging, time helpers, `protocol.c` with tests.
2. **Plain UDP loop:** sender sends numbered dummy packets on a timer; receiver counts and prints. Validates sockets and timers.
3. **Media + packetization:** `encode_variants.sh`; sender reads level 1 from memory, fragments frames; receiver reassembles and pipes to `ffplay`. Test on a clean loopback.
4. **Jitter buffer + playout:** fixed delay first, then adaptive; add the missing-frame policy.
5. **Netem:** loss, delay, jitter, reorder, bandwidth. Confirm the receiver survives each in isolation.
6. **Feedback channel:** receiver stats -> feedback packets -> sender logs RTT and loss.
7. **Pacer + AIMD:** rate follows feedback; verify the sawtooth on a bandwidth-limited link.
8. **ABR:** multi-level switching at GOP boundaries; verify seamless playback across switches.
9. **Metrics, plots, scenario scripts, README, final cleanup.**

Do not move to the next milestone until the current one is demonstrably working; most bugs in this kind of system come from skipping the isolated tests.

---

## 17. Pitfall checklist

- [ ] Serialize headers field by field (endianness, padding).
- [ ] Use serial-number arithmetic for `seq` and `frame_id`.
- [ ] Switch quality only at IDR boundaries; repeat SPS/PPS at every IDR.
- [ ] Pace packets; never burst a full I-frame.
- [ ] Rate-limit multiplicative decreases to once per RTT.
- [ ] Keep video on stdout and logs on stderr.
- [ ] Use `CLOCK_MONOTONIC`, not wall-clock time.
- [ ] Validate every incoming field (the network is untrusted); bound all `memcpy` sizes.
- [ ] Handle `EINTR` / `EAGAIN` on every socket call; check return values.
- [ ] Pre-allocate buffers; no `malloc` per packet.
- [ ] Clean shutdown: `BYE`, close sockets, flush CSVs, join the playout thread.

---

## 18. Possible extensions

- **NACK-based selective retransmission** for lost fragments of key frames (within the jitter-buffer deadline).
- **XOR parity FEC** (1 parity packet per N) to repair single losses without a round trip.
- **Delay-gradient congestion detection** (react to rising queuing delay before loss occurs, as in GCC/BBR-style schemes).
- **Receiver-driven ABR** (receiver picks the level and tells the sender).
- **Real-time encoding** with `libx264` and per-frame bitrate control instead of pre-encoded ladders.
- **Decode in-process** with `libavcodec` + SDL2 instead of piping to `ffplay`.
- **Multi-receiver** support with per-client congestion state.
