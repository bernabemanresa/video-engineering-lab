# Video Engineering with FFmpeg: Essential Survival Guide

**Focus:** Digital video fundamentals, compression, H.264/x264, and everyday FFmpeg commands.  
**Practice environment:** Ubuntu on WSL2, with FFmpeg, FFprobe, and FFplay.  
**Edition:** October 2026.

> **Goal:** Understand what happens to a video from input to playback, read basic x264 logs, make safe encoding decisions, and check the result. This is an introduction, not an encyclopaedia of codec options.

**How to study:** Read the chapters in order. Each new idea uses concepts from earlier chapters. Complete the short labs after Chapter 9. Each lab tests one idea; do not change several parameters at the same time.

---

## 1. Start with a picture

A **digital image** is a grid of **pixels**. A pixel is a position in the image that represents color or brightness. Each pixel is stored using numerical values called **samples**. The way these numbers are organized depends on the image format.

### 1.1 Resolution

The **resolution** tells us how many pixel positions an image has.

For example, **1920 × 1080** means 1,920 positions across and 1,080 down. That is approximately 2.07 million pixel positions.

Increasing resolution may allow more detail, but it also gives the encoder more information to process. A high-resolution video is not automatically sharp: the source may be blurry or have very little detail.

**Aspect ratio** describes the shape of the image. A common widescreen shape is **16:9**.

### 1.2 RGB and Y′CbCr

**RGB** describes colors using red, green, and blue components.

Many video codecs instead work with a representation called **Y′CbCr**:

- **Y′ (luma):** information related to picture brightness.
- **Cb and Cr (chroma):** information related to color differences.

People often say **YUV** when discussing digital video. Strictly speaking, *Y′CbCr* is the more accurate name in this context, but `yuv420p` is the actual FFmpeg pixel-format name.

The human visual system usually notices fine brightness detail more easily than equally fine color detail. Many video formats therefore keep full luma detail while storing less chroma detail.

**4:4:4**, **4:2:2**, and **4:2:0** describe how color information is sampled. In **4:2:0**, chroma has half the horizontal and half the vertical sample resolution of luma. It does **not** mean half of the entire picture is missing.

### 1.3 Bit depth and pixel format

**Bit depth** describes how many distinct numerical values a sample can represent:

- **8-bit:** 256 possible values per component.
- **10-bit:** 1,024 possible values per component.

A **pixel format** says how samples are arranged. Examples include `rgb24`, `yuv420p`, and `yuv420p10le`.

A 10-bit source may preserve smooth color or brightness changes better than an 8-bit version, depending on the rest of the pipeline. Changing an 8-bit file to 10-bit does not recover detail that was already lost.

### 1.4 Color is more than pixel format

A video also needs the correct interpretation of its colors and brightness. Key metadata includes:

- **Color primaries:** the red, green, and blue reference colors.
- **Transfer function:** how sample values relate to light.
- **Matrix coefficients:** the rules for converting RGB and Y′CbCr.
- **Range:** which numerical values are used for the useful signal (often called *limited* or *full* range).

Incorrect color interpretation can make a video look washed out, too dark, or wrongly colored even when encoding succeeds.

**Checkpoint:** Explain resolution, pixel format, 8-bit versus 10-bit, and the idea behind 4:2:0.

---

## 2. From pictures to video

A **frame** is one complete picture in a video sequence. A video displays frames in order so that we perceive movement.

### 2.1 Frame rate

**Frame rate** describes how often pictures are shown. **30 fps** means 30 frames per second. A ten-second video at a constant 30 fps normally has 300 frames.

Some common rates are 24, 25, 30, 50, and 60 fps, plus fractional values such as **30000/1001**, which is about 29.97 fps. Do not silently replace 29.97 with exactly 30 when precise timing matters.

- **CFR (Constant Frame Rate):** frames normally appear at regular intervals.
- **VFR (Variable Frame Rate):** the time between frames can change.

### 2.2 Timestamps

Real media uses **timestamps**, not just a frame counter.

A **PTS (Presentation Timestamp)** says when a frame should be displayed. Its numerical value uses a **time base**, which tells us the duration of one timestamp unit.

For example, with a time base of 1/90,000, a timestamp value of 90,000 represents one second.

We will introduce the **DTS (Decoding Timestamp)** later, after learning why some frames must be decoded in a different order from their display order.

### 2.3 Progressive and interlaced video

In **progressive** video, a frame normally represents a complete picture for one instant. A format label such as `1080p` uses `p` for progressive.

**Interlaced** video stores alternating groups of image lines, called **fields**. Fields may represent different moments in time. They need special treatment when converting to progressive video. Do not assume `1080i` is the same as `1080p`.

**Checkpoint:** Distinguish a frame from a pixel, and fps from a timestamp.

---

## 3. Codec, container, stream, and the processing pipeline

These words describe different jobs. Most beginner mistakes start by mixing them up.

### 3.1 Compression, encoding, and decoding

Uncompressed video needs a large amount of storage and network capacity.

**Compression** represents the information using fewer **bits**. A **bit** is 0 or 1; eight bits make one **byte**.

An **encoder** creates compressed data. A **decoder** reconstructs pictures from that data.

A **codec** defines rules for this process. **H.264/AVC** is a codec standard. **x264** is an encoder implementation of H.264; FFmpeg uses it through the name `libx264`.

The compressed data following the codec rules is called the **bitstream**.

### 3.2 Streams and containers

A movie may contain video, audio, and subtitles. Each separately encoded component is a **stream**.

A **container** stores one or more streams and the information needed to handle them together. **MP4** and **Matroska (`.mkv`)** are containers. They are not codecs.

For example, an MP4 file can contain H.264 video and AAC audio. **AAC** is an audio compression format.

A **muxer** combines encoded streams into a container. A **demuxer** opens a container and separates the encoded streams. The names describe packaging work, not video compression.

Now we can follow the complete path:

```text
Original frames
      |
      v
  Video encoder (x264) --> H.264 video stream ---+
                                                 |
  Audio encoder ----------> audio stream --------+--> muxer --> MP4 file
                                                                      |
                                                                      v
                                                                   demuxer
                                                                   /    \
                                                         video decoder  audio decoder
                                                               |              |
                                                            pictures         sound
                                                               \              /
                                                                   playback
```

A **player** reads the media, decodes its streams, keeps them in sync, and presents the result.

### 3.3 Transcoding versus remuxing

A **transcode** decodes input video and encodes it again, possibly after changing resolution or frame rate. A lossy re-encode can add visible damage.

A **remux** changes the container without recompressing the video. In FFmpeg, `-c copy` asks it to copy compatible compressed streams rather than decode and encode them.

A **filter** changes decoded frames, for example by scaling them. Filtering requires processing pictures, so you cannot combine an image-changing filter with direct video stream copy.

**Checkpoint:** Explain why “convert MP4 to H.264” is not a precise description. One is a container; the other is a codec.

---

## 4. Bitrate, size, and quality

**Bitrate** means bits used per second. Video rates are often written as kbps (thousands of bits per second) or Mbps (millions of bits per second).

A rough size estimate is:

```text
Video size in MB ≈ average video Mbps × seconds / 8
```

For example, 4 Mbps for 60 seconds is about **30 MB** of video data. Audio and container overhead add to the file size. `MB` uses decimal units; `MiB` uses binary units.

### 4.1 The basic trade-off

We care about three resources:

- **Bits:** storage and network traffic.
- **Quality:** how good the decoded pictures look.
- **Compute:** how long and how much CPU or other hardware the encoder needs.

At a similar quality level, a better encoding strategy may use fewer bits but more compute.

### 4.2 Lossy versus lossless

With **lossless compression**, decoding reproduces exactly the samples supplied to the encoder. **FFV1** is an example.

With **lossy compression**, decoding produces an approximation. H.264 is usually used in this mode.

A lossless file is not proof that the original picture was perfect: an earlier conversion or lossy encode may already have removed detail.

### 4.3 Average is not peak

Two files can have the same **average bitrate** but different short periods of high bitrate. This matters for playback and buffering.

**Checkpoint:** Explain why two videos at the same bitrate may look different, and why two videos at the same quality may have different sizes.

---

## 5. How H.264 saves bits

To understand an encoder log, we need only a few connected ideas:

**blocks → prediction → residual → transform → quantization → entropy coding**

The exact H.264 rules are more detailed, but this sequence gives a useful mental model.

### 5.1 Blocks and macroblocks

An encoder does not make one single decision for a complete frame. It divides pictures into regions called **blocks**.

In H.264, an important unit is the **macroblock**: a region covering **16 × 16 luma sample positions**. Smaller partitions can be used inside it.

Different parts of one frame can need different coding decisions. A smooth wall and detailed hair are not equally difficult.

### 5.2 Prediction inside a frame: intra

**Intra prediction** estimates a region using already reconstructed neighboring regions **in the same picture**.

For example, a smooth wall may be predicted from nearby pixels. Instead of sending every original value, the encoder can send a prediction choice plus any required correction.

### 5.3 Prediction between frames: inter

**Inter prediction** uses an already reconstructed **reference frame** to estimate the content of another frame.

Imagine a football moving ten pixels to the right. A useful prediction may come from a similar region in a reference frame, shifted to its new position.

The encoder can communicate that shift using a **motion vector**. The vector describes a useful prediction displacement; it is **not necessarily the physical motion of a real object**.

- **Motion estimation:** the encoder searches for a useful matching region and displacement.
- **Motion compensation:** the codec rebuilds the prediction using the chosen reference and motion information.

The encoder may test many possibilities. More search can improve compression, but it costs processing time.

### 5.4 Residual: what the prediction missed

Prediction is rarely perfect. The difference between the original sample and its prediction is the **residual**.

```text
Original values:    120  121  120  122
Predicted values:   120  120  120  120
Residual values:      0    1    0    2
```

Good prediction often produces smaller, easier-to-code residuals. Motion alone is not enough to predict bitrate: what matters is also **how well the motion can be predicted**.

You may hear **residual energy**. It refers to the overall size of the remaining differences, often measured using squared values. You do not need the equation to understand its role.

### 5.5 Transform, quantization, and entropy coding

A **transform** reorganizes a block of residual values into numbers called **coefficients**. These can be easier to compress than the original differences.

**Quantization** reduces the precision of coefficient values. It can turn small values into zero and save many bits. It is a major source of intentional information loss.

**Entropy coding** then represents the remaining symbols using fewer bits, without adding further quantization error. **CABAC** is one entropy-coding method used by H.264.

The decoder performs the corresponding reconstruction steps. It cannot restore precision discarded by lossy quantization.

### 5.6 Skip: a successful prediction

A **skip macroblock** is a coding mode in which the decoder can infer prediction information without receiving an explicitly coded residual for that macroblock.

**Skip does not mean that pixels are ignored or removed.** The decoder still reconstructs the picture region.

If x264 reports `skip:84.9%`, most macroblocks in that picture class used skip mode. The rest can still require many bits, so high skip does not guarantee a small file.

### 5.7 I-, P-, and B-pictures

These letters tell us what kinds of prediction are allowed:

| Picture type | Main idea | Useful fact |
|---|---|---|
| **I** | Uses prediction within its own picture | Does not require another picture for temporal prediction |
| **P** | Can use reference pictures through one reference list | May also contain intra-coded regions |
| **B** | Can use two reference lists | Often provides better compression, but may require reordering |

An **I-picture is not an uncompressed frame**. It can still be heavily compressed and quantized.

The reference structure determines which pictures must be available before others can be reconstructed.

### 5.8 GOP and access points

A **GOP (Group of Pictures)** is a group organized around picture prediction and access points.

A simplified *display-order* example:

```text
I  B  B  P  B  B  P  B  B  I ...
```

An **IDR (Instantaneous Decoder Refresh)** is a special H.264 access point that resets reference dependencies according to codec rules. **Not every I-picture is an IDR.**

- **Closed GOP:** decoding dependencies stay within the GOP boundary in the practical access-point sense.
- **Open GOP:** pictures may use references across GOP boundaries.

The x264 parameter `keyint=250` describes a maximum keyframe interval of 250 frames, not an interval of exactly 250 frames. At 30 fps, 250 frames last about 8.33 seconds.

**GOP length affects random access and recovery.** It does not by itself determine end-to-end latency.

### 5.9 Why DTS exists

We already know **PTS**, the display timestamp. B-pictures may depend on reference pictures that appear **later** on screen.

Those future-display references must be decoded first. Therefore, **decode order** may differ from **display order**.

**DTS (Decoding Timestamp)** says when encoded data is due for decoding. PTS says when the resulting picture should be shown.

This distinction matters when inspecting compressed packets with FFprobe.

**Checkpoint:** Explain block, intra, inter, motion vector, residual, quantization, skip, I/P/B, GOP, and IDR. If you can explain the chain, most of the x264 log becomes much less mysterious.

---

## 6. QP, CRF, bitrate control, and presets

You now know that quantization removes precision. Next, learn how the encoder decides **how much** precision to keep.

### 6.1 QP

**QP (Quantization Parameter)** controls the strength of quantization. In ordinary H.264 8-bit discussions:

- **Lower QP:** generally less loss and more bits.
- **Higher QP:** generally more loss and fewer bits.

QP is **not a direct quality score**. The same QP can produce very different visual results in different scenes.

An x264 log may report different average QPs for I-, P-, and B-pictures.

### 6.2 Rate control (RC)

**Rate control** is the logic that decides how to spend bits over time and across different parts of the video.

Three common approaches are:

| Mode | You mainly request | What you must understand |
|---|---|---|
| **CRF** | A quality–size policy | Output bitrate and size are not fixed |
| **ABR** | An average bitrate target | Quality changes as necessary to approach the target |
| **Constant QP** | A quantization level | Neither bitrate nor perceived quality is constant |

**CRF (Constant Rate Factor)** is especially common with x264. A command using `-crf 18` asks the encoder to follow a relatively high-quality policy. A higher value, such as 28, generally produces a smaller but more degraded result for the same source and settings.

**CRF 18 is not VMAF 18, not QP 18, and not an 18 Mbps target.**

**ABR (Average Bitrate)** uses an option such as `-b:v 1000k` to aim for about 1,000 kilobits per second on average. Very short clips can miss the target.

### 6.3 VBV: controlling bursts

**VBV (Video Buffering Verifier)** is a model used to constrain the rate at which compressed data can fill a decoder's hypothetical buffer.

For x264, FFmpeg options include:

- `-maxrate 1500k`: rate limit in the VBV model.
- `-bufsize 3000k`: capacity of the VBV model's buffer, expressed in bits.

VBV is important when playback or delivery cannot tolerate unconstrained bitrate peaks.

**CBR (Constant Bitrate)** is an overloaded term. Setting only `-b:v` does **not** guarantee that every second of the file uses the same number of bits. Strictly constant transport rates may require additional behavior.

### 6.4 Presets and CPU time

An encoder **preset** is a bundle of decisions about how much work the encoder will do.

Examples in x264 are `ultrafast`, `veryfast`, `medium`, `slow`, and `veryslow`.

At similar output quality, a slower preset can often save bitrate, but it consumes more CPU and takes longer. A preset is **not** itself a quality score.

### 6.5 AQ, MB-tree, and lookahead

These advanced names appear frequently in logs. For a first reading, remember the purpose of each:

- **AQ (Adaptive Quantization):** changes bit allocation across regions of a picture. Some regions benefit from different quantization decisions.
- **MB-tree (Macroblock Tree):** uses information about how blocks will be referenced in later pictures to guide bit allocation.
- **Lookahead:** lets the encoder inspect upcoming input pictures before deciding some frame types and bit allocation.
- **Motion search (`me`, `subme`):** controls parts of the search and refinement work used for motion prediction.
- **Reference count (`ref`):** changes how many reconstructed reference pictures the encoder can consider.
- **Deblocking:** a reconstruction filter that helps reduce block-boundary artifacts.

For example:

```text
rc=crf crf=18.0 aq=1:1.00 mbtree=1
me=hex subme=7 ref=3 rc_lookahead=40
```

This reports a CRF policy, adaptive quantization, MB-tree enabled, a motion search configuration, three reference pictures, and 40-frame lookahead. It does not prove that the chosen settings are optimal.

**Do not tune these advanced settings first.** Start with correct input interpretation, an appropriate preset, a chosen rate-control mode, and quality measurement.

**Checkpoint:** Explain why CRF, QP, bitrate, and VMAF are four different concepts.

---

## 7. FFmpeg: the minimum you must understand

Three programs form the basic toolkit:

- **FFmpeg:** reads media, processes it, and writes output.
- **FFprobe:** inspects files, streams, frames, and timestamps.
- **FFplay:** plays media for a quick visual check.

### 7.1 A command has a direction

A typical command looks like this:

```bash
ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 23 output.mp4
```

Read it left to right:

1. `ffmpeg`: run the program.
2. `-i input.mp4`: open the input file.
3. `-c:v libx264`: encode the output video using x264.
4. `-preset medium`: select an encoder speed/efficiency preset.
5. `-crf 23`: select a CRF rate-control policy.
6. `output.mp4`: write the result as an MP4 file.

**Order matters.** FFmpeg options may apply to an input or to an output depending on where they appear.

### 7.2 Streams and `-map`

A file may have more than one video, audio, or subtitle stream.

`-map 0:v:0` means **take the first video stream from input 0**. `-an` means disable audio in the output.

For controlled video exercises, selecting the stream explicitly helps prevent accidental changes to what is being compared.

### 7.3 Stream copy versus a filter

Use stream copy when you want to keep compatible compressed streams:

```bash
ffmpeg -i input.mp4 -c copy output.mkv
```

Use a filter such as `-vf scale=640:360` only when you need to modify decoded frames. Changing pixels requires a new encode if you want another compressed video file.

### 7.4 How to interpret execution

FFmpeg typically reports:

```text
frame=180 fps=120 time=00:00:05.93 speed=4.0x
```

- `frame`: processed video frames.
- `fps`: processing throughput, **not** necessarily the source frame rate.
- `time`: processed media timeline position.
- `speed`: processing speed relative to media time. `4.0x` means around four seconds of video processed per second of wall-clock time.

The displayed progress values can differ slightly from the final duration due to timestamps and how progress is reported.

Check the program's **exit status**, not just the existence of an output file. In a shell, `echo $?` displays the previous command's exit status; zero normally means success.

### 7.5 The compressed packet

FFprobe may show **packets**. A packet is a unit of encoded data passed between parts of the processing pipeline. A packet is **not** the same thing as a displayed pixel block, and you should not assume it always corresponds to one complete frame.

When inspecting frames, use `-show_frames`; when inspecting packets, use `-show_packets`. This helps keep decoded picture concepts separate from encoded data concepts.

**Checkpoint:** Explain what `-i`, `-c:v`, `-crf`, `-preset`, `-map`, `-vf`, and `-c copy` do.

---

## 8. Reading an actual x264 result

This chapter uses the ten-second Full HD test discussed during the onboarding. It was encoded with `libx264`, `medium`, and CRF 18.

### 8.1 Output properties

```text
Video: h264, yuv420p(tv, progressive), 1920x1080
30 fps
```

Read this as: H.264 compressed video, 4:2:0 eight-bit samples, limited-range signal, progressive pictures, 1920 × 1080, at 30 fps.

The log also reported:

```text
profile High, level 4.0, 4:2:0, 8-bit
```

A **profile** identifies a set of H.264 coding capabilities. A **level** places limits on decoding resources such as frame size, processing demand, and reference memory. Neither is a quality score.

### 8.2 Timing, size, and speed

```text
frame=300 fps=142
Lsize=12176KiB
time=00:00:09.93
bitrate=10041.7kbits/s
speed=4.69x elapsed=0:00:02.11
```

The job encoded 300 frames in about 2.11 seconds of real time: faster than playback. The final file was about 11.9 MiB.

The output bitrate was close to 10 Mbps. That was a **result of CRF 18**, not a bitrate the user requested.

The final x264 line said `kb/s:9970.73`. This is slightly different from FFmpeg's container-level calculation because the two statistics do not count and time data in exactly the same way.

### 8.3 Picture types and average QP

```text
frame I:2     Avg QP: 6.52  size: 64016
frame P:135   Avg QP:19.63  size: 46069
frame B:163   Avg QP:24.63  size: 37522
```

There are 2 I-pictures, 135 P-pictures, and 163 B-pictures.

Each class has a different **average QP**. The B-pictures were quantized more strongly on average. You cannot convert these QP values directly into VMAF scores or use them to rank the visual quality of individual pictures.

The `size` values are **average encoded picture sizes in bytes** for each class.

### 8.4 Skip percentages

```text
mb P ... skip:84.9%
mb B ... skip:86.4%
```

Most macroblocks in the P- and B-picture classes used skip mode. Their reconstruction needed no explicitly coded residual at those macroblocks.

This shows that prediction was effective across many blocks. It does **not** mean that the other blocks were cheap or that the whole file should be small.

### 8.5 Other settings you can now recognize

```text
cabac=1 ref=3 me=hex subme=7
bframes=3 keyint=250
rc=crf crf=18.0
aq=1:1.00 mbtree=1
```

- `cabac=1`: entropy coding uses CABAC.
- `ref=3`: reference picture configuration.
- `me=hex`, `subme=7`: motion search and refinement settings.
- `bframes=3`: maximum consecutive B-picture setting.
- `keyint=250`: maximum keyframe interval.
- `rc=crf`, `crf=18.0`: rate-control policy.
- `aq=1:1.00`: adaptive quantization mode and strength.
- `mbtree=1`: MB-tree enabled.

**What the log cannot tell you:** whether the video looks good to a viewer or whether the settings are best for your use case. For that, inspect the pictures and measure quality against a known reference.

---

## 9. Checking quality without guessing

A compressed file being smaller does not prove it is better.

### 9.1 Reference and encoded output

To compare image quality, use the same visual content in two forms:

- **Reference:** the picture data before the lossy encoding being tested.
- **Encoded output:** the decoded result of that encoding.

Both must have the same content at matching times. Unexpected cropping, frame-rate conversion, color changes, or resolution mismatch can invalidate a metric comparison.

### 9.2 Three quality metrics

- **PSNR (Peak Signal-to-Noise Ratio):** compares numerical sample error. Higher normally means closer to the reference.
- **SSIM (Structural Similarity):** compares aspects of image structure. Higher normally means more similar.
- **VMAF (Video Multi-Method Assessment Fusion):** a model trained to estimate aspects of perceived video quality from a reference and distorted picture sequence.

None is perfect. A high score does not guarantee freedom from banding, blur, flicker, or visible motion artifacts.

**Banding** means that a smooth change of brightness or color looks like separate steps. **Blocking** means visible block-like boundaries. **Ringing** appears as unwanted patterns around sharp edges.

For a first lab, PSNR is enough to learn how objective quality comparisons work. VMAF belongs in a deeper document after you can confidently inspect the input and output.

### 9.3 A useful first engineering decision

When comparing two H.264 settings:

1. Use the same reference and keep resolution and frame rate unchanged.
2. Change **one setting**.
3. Record actual bitrate, file size, encode time, and a quality result.
4. Look at the pictures, especially difficult moments.
5. Repeat if the difference is small.

A fair experiment is more important than an impressive single number.

---

## 10. Ten small FFmpeg labs

Each lab has **one question** to answer. The clips are deliberately short. Run the labs in this order; later labs reuse files created earlier.

### Preparation: install the tools

In Ubuntu on WSL2:

```bash
sudo apt update
sudo apt install -y ffmpeg

mkdir -p ~/video-survival
cd ~/video-survival
ffmpeg -version | head -n 1
```

Check that `libx264` and `ffv1` are available:

```bash
ffmpeg -hide_banner -encoders | grep -E 'libx264|ffv1'
```

If `ffplay` does not open a window, graphical WSL support may be missing. The other exercises can still run.

### Lab 1 — Create an input video

**Question:** How do we get a short, repeatable input file?

Use FFmpeg's built-in test pattern. `lavfi` lets a filter provide frames as input; `testsrc2` is a moving test pattern. This is the **only exercise about making the input**.

```bash
ffmpeg -y -hide_banner \
  -f lavfi -i 'testsrc2=size=640x360:rate=30:duration=6' \
  -pix_fmt yuv420p -c:v ffv1 source.mkv
```

**Observe:** You now have a six-second lossless-coded reference (`source.mkv`) for every later lab. FFV1 preserves the samples it receives. The file may be large.

### Lab 2 — Play the video

**Question:** Can we decode and display the file?

```bash
ffplay -autoexit source.mkv
```

**Observe:** FFplay reads the container, decodes its video stream, and shows the frames. No new encoded output is created.

### Lab 3 — Inspect the video

**Question:** What is inside the file?

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=codec_name,width,height,pix_fmt,avg_frame_rate \
  -of default=noprint_wrappers=1 source.mkv
```

**Observe:** Codec `ffv1`, resolution `640×360`, pixel format `yuv420p`, and frame rate `30/1`. The extension `.mkv` identifies the container, not the codec.

### Lab 4 — Encode H.264

**Question:** What does a basic H.264 transcode look like?

```bash
ffmpeg -y -hide_banner -i source.mkv \
  -c:v libx264 -preset medium -crf 23 -pix_fmt yuv420p \
  h264.mp4
```

**Observe:** This decodes the FFV1 input and encodes it using H.264. The output container is MP4.

### Lab 5 — Remux without re-encoding

**Question:** Can the container change while the compressed video stays the same?

```bash
ffmpeg -y -hide_banner -i h264.mp4 -c copy h264.mkv
```

**Observe:** The file changes from MP4 to MKV, but the video codec remains H.264. No new lossy encode is requested.

### Lab 6 — Change only CRF

**Question:** What happens to file size when we change CRF?

```bash
ffmpeg -y -hide_banner -i source.mkv \
  -c:v libx264 -preset medium -crf 18 crf18.mp4

ffmpeg -y -hide_banner -i source.mkv \
  -c:v libx264 -preset medium -crf 30 crf30.mp4

ls -lh crf18.mp4 crf30.mp4
```

**Observe:** The lower CRF generally produces a larger file with less distortion. Compare the files visually, but do not assume file size is a quality metric.

### Lab 7 — See I-, P-, and B-pictures

**Question:** Does the encoder use different picture types?

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries frame=pict_type -of csv=p=0 \
  crf18.mp4 | sort | uniq -c
```

**Observe:** The counts of I, P, and B pictures add up to the decoded picture count. Their relative proportions depend on content and encoder decisions.

### Lab 8 — Control the keyframe interval

**Question:** How does GOP configuration change the spacing of access points?

```bash
ffmpeg -y -hide_banner -i source.mkv \
  -c:v libx264 -preset medium -crf 23 \
  -g 30 -keyint_min 30 -sc_threshold 0 \
  -x264-params open-gop=0 \
  gop30.mp4

ffprobe -v error -select_streams v:0 -skip_frame nokey \
  -show_entries frame=best_effort_timestamp_time \
  -of csv=p=0 gop30.mp4
```

**Observe:** With a 30 fps input, a 30-frame keyframe interval corresponds to approximately one second. Check the actual output rather than trusting a requested setting blindly.

### Lab 9 — Request an average bitrate

**Question:** What changes when we request an average bitrate instead of CRF?

```bash
ffmpeg -y -hide_banner -i source.mkv \
  -c:v libx264 -preset medium -b:v 800k \
  abr800.mp4

ffprobe -v error -select_streams v:0 \
  -show_entries stream=bit_rate -of default=noprint_wrappers=1 \
  abr800.mp4
```

**Observe:** `-b:v 800k` requests an average video bitrate of about 800 kbps. The achieved bitrate may differ, especially on a short, low-complexity clip.

### Lab 10 — Measure one quality difference

**Question:** Can we measure how much the encode changed the reference?

```bash
ffmpeg -hide_banner -i source.mkv -i crf30.mp4 \
  -filter_complex '[0:v][1:v]psnr' \
  -f null -
```

**Observe:** Look for the final `PSNR` statistics. PSNR measures numerical similarity; it is not a promise of perceived quality. The exercise compares files with matching frames, resolution, and pixel format.

**Stop here.** You now have enough practice to read ordinary FFmpeg commands, create an H.264 output, recognize the core x264 concepts, and run a basic comparison. Avoid tuning more parameters until these ten exercises make sense.

---

## 11. The practical H.264 survival checklist

Before running an encode:

1. **Inspect the source:** codec, resolution, pixel format, frame rate, and timestamps.
2. **Choose the output codec deliberately:** `libx264` for software H.264 encoding.
3. **Know the destination:** MP4, MKV, or another supported container.
4. **Choose one rate-control goal:** CRF for a quality-oriented test; target bitrate for a rate budget.
5. **Choose a preset:** start with `medium`; compare with `slow` only when it is worth the extra time.
6. **Check compatibility:** pixel format, H.264 profile/level requirements, and the target player.
7. **Inspect the result:** frame types, bitrate, file size, and playback.
8. **Measure before claiming an improvement:** same reference, same conditions, and visual review.

### What can wait for another document?

You do not need to master these topics to survive your first days with FFmpeg:

- The full H.264 bitstream syntax, NAL-unit types, or entropy-coding mathematics.
- Every x264 parameter, custom quantization matrix, or rate-distortion algorithm.
- Detailed VBV/HRD buffer compliance.
- HEVC, AV1, hardware encoder tuning, or cross-codec benchmarks.
- Advanced VMAF models, scene-aware encoding, and bitrate optimization.
- HLS/DASH packaging, CDN design, or live-streaming latency engineering.

Those subjects are important, but each deserves its own focused guide.

---

## 12. Quick glossary

Use this as a reminder **after** reading the corresponding explanations.

| Term | Meaning |
|---|---|
| **Pixel** | A position in a digital picture. |
| **Sample** | A stored numerical value used to represent picture data. |
| **Frame** | One complete displayed picture. |
| **FPS** | Frames per second. |
| **PTS** | Timestamp for presentation/display. |
| **DTS** | Timestamp for decoding compressed data. |
| **Codec** | Rules for representing and reconstructing compressed media. |
| **Encoder** | Produces compressed data. |
| **Decoder** | Reconstructs data for playback. |
| **Stream** | A separately encoded media component, such as video or audio. |
| **Container** | A file structure that holds streams and timing information. |
| **Muxer** | Writes streams into a container. |
| **Demuxer** | Reads separate streams from a container. |
| **Bitrate** | Bits used per second. |
| **Lossy** | Reconstruction may differ from the original. |
| **Lossless** | Reconstruction preserves the samples given to the encoder. |
| **Block / macroblock** | Region within a frame that receives coding decisions. |
| **Intra** | Prediction from the same picture. |
| **Inter** | Prediction using reference pictures. |
| **Motion vector** | Displacement used to build an inter prediction. |
| **Residual** | Difference between original data and its prediction. |
| **Quantization** | Reduction in precision used to save bits. |
| **QP** | Parameter controlling quantization strength. |
| **Skip** | Prediction mode without explicitly coded residual for that macroblock. |
| **I/P/B** | Picture types with different allowed prediction structures. |
| **GOP** | Group of pictures organized around prediction and access points. |
| **IDR** | H.264 access point that resets reference dependencies. |
| **CRF** | x264 quality–size rate-control policy. |
| **ABR** | Average bitrate targeting. |
| **VBV** | Rate/buffer constraint model. |
| **Preset** | Collection of encoder settings controlling speed/efficiency. |
| **AQ** | Adaptive Quantization. |
| **MB-tree** | Block-level bit allocation informed by future reference value. |
| **PSNR** | Numerical signal similarity metric. |
| **SSIM** | Structural image similarity metric. |
| **VMAF** | Model-based assessment of perceptual video quality. |

---

## Technical references

These are primary references for further study. FFmpeg behavior may depend on the installed version and build configuration.

- FFmpeg Project. (n.d.). *FFmpeg documentation*. https://ffmpeg.org/ffmpeg.html
- FFmpeg Project. (n.d.). *FFprobe documentation*. https://ffmpeg.org/ffprobe.html
- FFmpeg Project. (n.d.). *FFplay documentation*. https://ffmpeg.org/ffplay.html
- FFmpeg Project. (n.d.). *FFmpeg filters documentation*. https://ffmpeg.org/ffmpeg-filters.html
- FFmpeg Project. (n.d.). *FFmpeg codecs documentation*. https://ffmpeg.org/ffmpeg-codecs.html
- VideoLAN. (n.d.). *x264: H.264/MPEG-4 AVC encoder*. https://www.videolan.org/developers/x264.html

---

## AI assistance and document attribution

This document was prepared and reviewed with the assistance of **ChatGPT (OpenAI, GPT-6)** using the author's learning goals and technical instructions.

- **Tool:** ChatGPT
- **Model:** GPT-6
- **Developer:** OpenAI
- **Revision date:** October 18, 2026
- **Website:** https://chatgpt.com/

AI-assisted technical explanations may contain errors. Verify important settings and operational decisions against official documentation and real tests.
