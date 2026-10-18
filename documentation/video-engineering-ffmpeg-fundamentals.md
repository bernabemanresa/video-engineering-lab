# Video Engineering with FFmpeg: An Onboarding Guide

**Edition:** October 2026 — technically reviewed  
**Practice environment:** FFmpeg, FFprobe, and FFplay. Most labs use modest CPU and disk resources.  
**Main codec:** H.264/AVC and the x264 encoder. HEVC/H.265 and AV1 are introduced and compared.  
**Learning method:** Explain the problem first. Then learn the tool. Finally, test it with real commands.

> **Purpose.** After this guide, you should be able to read an FFmpeg command or an x264 log and explain what happens to the video. You should understand the input, the processing steps, the compression choices, the resulting quality, the time and CPU cost, and how to check the result. You do not need previous video knowledge or advanced mathematics.

> **Scope.** This is a personal learning guide. It aims to teach useful engineering principles. It is not a replacement for official codec standards, production testing, or a professional certification.

## How to use this guide

Read the chapters in order. Most video ideas depend on earlier ideas. For example, to understand a `skip` macroblock, you first need to understand a frame, a block, prediction, and residual data.

1. **First pass — Chapters 1–9:** Learn what a video signal is. Complete Labs 1–5.
2. **Second pass — Chapters 10–17:** Learn how H.264 works. Complete Labs 6–11.
3. **Third pass — Chapters 18–25:** Learn measurement, streaming, and operations. Complete Labs 12–18.
4. **Review:** Read the real x264 log, answer the questions, and use the troubleshooting and glossary sections.

A good engineer does not need to remember every encoder parameter. A good engineer knows **what problem a parameter solves, what it costs, and how to test its effect**.

Some commands use `-y`. This means FFmpeg can overwrite an existing output file.

### Contents

- **Part I — Video as a signal:** 1. System overview; 2. Pixels and color; 3. Frames and time; 4. Codecs and containers; 5. Uncompressed, lossless, and synthetic video.
- **Part II — How compression works:** 6. Redundancy and information; 7. Blocks; 8. Prediction and motion; 9. I/P/B and GOP; 10. Residuals, transforms, and quantization; 11. QP and rate control; 12. Advanced x264 tools; 13. H.264 bitstream and decoding; 14. Artifacts; 15. Profiles and compatibility.
- **Part III — FFmpeg engineering:** 16. Commands and pipeline; 17. FFprobe; 18. Filters and color; 19. Presets and performance; 20. HEVC and AV1.
- **Part IV — Quality and delivery:** 21. Video quality metrics; 22. Rate–distortion experiments; 23. Audio and synchronization; 24. HLS, DASH, and CDN; 25. Live and low-latency streaming.
- **Part V — Guided labs:** Labs 1–18.
- **Appendices:** A. A real x264 log; B. Troubleshooting; C. Learning checks; D. Glossary; E. Quick-reference commands.

---

# Part I. Video as a signal

## 1. The video system: from a camera to a screen

A video system receives a sequence of images. It may also receive audio, subtitles, and other data. Its job is to deliver that content at a useful quality, size, speed, and cost.

```text
Camera / graphics renderer / synthetic video source
        |
        v
Original frames + timing + color information
        |
        v
Preprocessing (resize, frame rate, colors, filters)
        |
        v
ENCODER (H.264, HEVC, AV1, ...)
        |  predict -> calculate differences -> quantize -> encode symbols
        v
Compressed BITSTREAM
        |
        v
MUXER (MP4, MKV, MPEG-TS, fragmented MP4, ...)
        |
        v
File / origin server / CDN / network / live delivery
        |
        v
DEMUXER -> DECODER -> reconstructed frames -> screen
```

The main words are easy to confuse:

- **Frame:** one video image at a given moment.
- **Codec:** rules for representing compressed video and rebuilding its images. H.264 is a standard. `libx264` is an encoder that implements it.
- **Bitstream:** the encoded data that follows the codec rules.
- **Container:** a file format that holds video, audio, timing information, and metadata. Common examples are MP4 and Matroska (MKV).
- **Player:** software that reads the container, decodes streams, keeps them in sync, and displays or plays them. `ffplay` is a basic test player.

**Simple comparison:** Think of a book. The story is the content. A compression method is a way to store its text using fewer bytes. The book file format is the container. The reading application shows it to you. Converting a PDF into an EPUB does not mean rewriting the story. In the same way, moving a video stream from MP4 to MKV does not always need video re-encoding.

### 1.1 Six questions a video engineer should ask

1. What is the input, and how good is it?
2. Which codec, format, and features does the target device need?
3. What limits do we have for bitrate, CPU, energy, storage, and latency?
4. What quality loss is acceptable, and what defects are not acceptable?
5. How will we measure results fairly?
6. How will we detect, reproduce, and fix failures in production?

### 1.2 Encode, decode, transcode, and remux

**Encode** means converting image data into a compressed bitstream. **Decode** means rebuilding images from that bitstream. **Transcode** usually means decoding a compressed input and encoding it again, perhaps with a different codec, resolution, or bitrate. **Remux** means changing the container without changing compatible compressed streams. In FFmpeg, `-c copy` is used for stream copy.

**Key idea:** An `.mp4` file does not tell you everything about the video. Inspect the streams, codecs, pixel format, color information, and timestamps.

## 2. Pixels, resolution, and color

A digital image is a grid of samples. A 1920 × 1080 image contains 2,073,600 pixel positions. The samples may be stored in RGB or another format. They are not always stored as three RGB bytes per pixel.

### 2.1 Resolution and aspect ratio

Common video sizes are:

| Common name | Stored size | Pixel positions per frame |
|---|---:|---:|
| 360p | 640 × 360 | 230,400 |
| 720p | 1280 × 720 | 921,600 |
| 1080p | 1920 × 1080 | 2,073,600 |
| 2160p / UHD | 3840 × 2160 | 8,294,400 |

The `p` means **progressive scanning**, not a fixed width. Doubling both image dimensions creates four times as many pixels. This often increases processing cost. It does **not** mean a codec always needs four times the bitrate for the same visible quality.

**DAR (Display Aspect Ratio)** describes the shape shown on screen. For example, 16:9. **SAR (Sample Aspect Ratio)** describes the shape of a stored pixel sample. Most modern web video uses SAR 1:1, but not every video does.

### 2.2 RGB, Y′CbCr, and 4:2:0

RGB stores red, green, and blue components. Most digital delivery formats instead use **Y′CbCr**. `Y′` is **luma**: a coded measure related to image brightness. It is not exactly physical luminance. `Cb` and `Cr` describe color differences. People often say “YUV” when they mean digital Y′CbCr.

The eye is often more sensitive to small changes in luma detail than to small changes in color detail. Codecs use this property by storing less **chroma** detail:

- **4:4:4:** no chroma resolution reduction.
- **4:2:2:** chroma has half the horizontal sampling rate of luma.
- **4:2:0:** in the usual progressive arrangement, chroma has half the horizontal and vertical sampling rates.

```text
4:2:0 luma samples:          Chroma samples (conceptual):
Y Y Y Y                       C . C .
Y Y Y Y                       . . . .
Y Y Y Y                       C . C .
Y Y Y Y                       . . . .
```

For 8-bit **planar YUV420p**, storage is about **1.5 bytes per pixel**. For RGB24, it is 3 bytes per pixel. At 1920 × 1080, one YUV420p frame uses `1920 × 1080 × 1.5 = 3,110,400` bytes. This is the size **before temporal video compression**. For common even dimensions, the calculation is exact.

Image appearance also depends on **color primaries**, **transfer function**, **matrix coefficients**, and **signal range**. Wrong interpretation can change brightness, black levels, or colors even when a codec works correctly.

### 2.3 Bit depth: 8, 10, and 12 bits

An 8-bit sample can represent 256 values. A 10-bit sample can represent 1,024 values. More levels help describe smooth color changes and can reduce visible bands in gradients. But the full pipeline must keep that precision.

**Important:** 10-bit does not automatically mean HDR. A video can be 10-bit SDR. An encoder's internal calculation precision may also be different from its input pixel format.

### 2.4 Limited range, full range, and HDR

A typical 8-bit limited-range video uses nominal luma values 16–235 and chroma values 16–240. Full-range video usually uses 0–255. These are signal definitions. They are not artistic brightness settings.

In HD SDR, **BT.709** is common. In wide-color-gamut and UHD systems, **BT.2020** may be used. HDR can use transfer functions such as **PQ** or **HLG**. Adding an HDR label to SDR pixels does not turn them into true HDR content.

**Key idea:** Before measuring codec quality, ensure the reference and test video use compatible resolution, timing, pixel format, and color handling.

## 3. Frames, frame rate, timestamps, and movement

Video is a sequence of pictures **plus the time at which each picture should be shown**. A 10-second constant-frame-rate video at 30 fps usually has 300 frames. Other common rates are 24, 25, 50, 60, and fractional rates such as 23.976, 29.97, and 59.94.

### 3.1 Frame rates can be fractions

29.97 fps often means exactly `30000/1001`, not 30. This small difference matters when matching audio, creating GOPs, or cutting long videos. Use the rational value, not a rounded number, for time-sensitive work.

### 3.2 CFR and VFR

**CFR (Constant Frame Rate)** means regular presentation intervals. **VFR (Variable Frame Rate)** means the time between frames can change. Screen recordings and phone videos may use VFR. Converting VFR to CFR may duplicate or drop frames. Do it only when the destination requires it, and measure the result.

### 3.3 PTS and DTS

- **PTS (Presentation Timestamp):** when a frame is shown.
- **DTS (Decoding Timestamp):** when a compressed picture must be decoded.
- **Time base:** the unit used to store timestamp numbers.

B-frames can depend on a frame that will be shown later. That reference may need to be decoded first. So decoding order and display order can differ:

```text
Display order:   I0  B1  B2  P3
Decode order:    I0  P3  B1  B2   (teaching example)
```

Real patterns depend on reference choices and codec rules. Do not assume every B-frame follows exactly this example.

### 3.4 Visible motion is not the same as coding difficulty

Movement may be caused by a running person, camera panning, zoom, changing light, or a scene cut. A fast moving background can still be easy to predict when its structure stays similar. Changing random noise can be hard to predict even if a viewer sees no moving object.

### 3.5 Progressive and interlaced video

A progressive frame contains an image for one time position. Traditional interlaced video uses separate **fields** with alternating lines, often captured at different times. Wrong processing can create comb-like edges on moving objects. When handling 1080i, 576i, or 480i, check the field order and decide whether deinterlacing is needed. Fifty fields per second are not the same thing as fifty full progressive frames per second.

**Key idea:** Use timestamps when exact timing matters. `frame_count / fps` is not enough for every VFR stream.

## 4. Codec, encoder, decoder, container, and bitrate

### 4.1 Using the right names

A video standard describes how a conforming bitstream is decoded. Different companies can make compatible encoders and decoders. `libx264` is a software H.264 encoder. `h264_nvenc` is an NVIDIA hardware encoding path. HEVC/H.265 and AV1 are different compression formats. None of these names means “MP4.”

An MP4 container may hold different supported codecs. Matroska supports many stream types. Always inspect what is inside the file.

### 4.2 What bitrate means

**Bitrate** is the number of bits used per second. **1 Mbps** is about one million bits per second, not one million bytes. Eight bits make one byte.

A useful estimate is:

```text
Video megabytes ≈ average Mbps × seconds / 8
```

Example: 4 Mbps for 60 seconds is about 30 MB of video data. Container overhead and audio add to the total. Distinguish **MB** (decimal) from **MiB** (binary).

Bitrate can be measured for the complete file, only video, only audio, or a short interval. These are different measurements. Two videos with the same average bitrate can have very different bitrate peaks.

### 4.3 Size, quality, and processing cost

A low bitrate usually helps reduce storage and delivery cost. But very low bitrate can damage visible quality. A slower encoder may produce a better result for the same bitrate, but needs more CPU time. A good design balances **quality, bits, compatibility, processing time, and latency**.

### 4.4 Stream copy, transcode, and remux

`-c copy` keeps encoded stream data instead of decoding and re-encoding it. This is fast and normally does not introduce a new generation of codec quality loss. It cannot perform changes that require modifying image pixels, such as scaling. A container change also must be compatible with the input streams and destination format.

## 5. Uncompressed, lossless, and synthetic video

These words describe different properties:

- **Uncompressed / raw:** samples are stored without a general video compression method. Raw YUV often has no internal width, height, fps, or color metadata.
- **Lossless coding:** the decoder reproduces the samples that were given to the encoder, without sample changes. **FFV1** is a useful lossless codec for labs.
- **Lossy coding:** the decoder reconstructs an approximation. H.264, HEVC, and AV1 are often used in lossy modes.
- **Synthetic video:** software generates the images. Synthetic video can be uncompressed, lossless-coded, or lossy-coded.

### 5.1 FFmpeg synthetic sources (`lavfi`)

The input format `lavfi` allows FFmpeg filters to create their own frames. `testsrc2` generates moving test patterns. `color` makes a solid image. Other filters can generate gradients, bars, noise, and moving shapes. This is ideal for repeatable experiments.

```bash
ffmpeg -f lavfi -i 'testsrc2=size=640x360:rate=30:duration=6' \
  -c:v ffv1 reference.mkv
```

The source creates frames. FFV1 stores them without further coding loss. Matroska holds the resulting bitstream and its timing.

### 5.2 What “lossless” really guarantees

Lossless coding preserves the **samples provided to that encoder**. If you first convert RGB 4:4:4 to YUV420p, you may already have lost color detail. Using FFV1 after the conversion cannot restore the original RGB information. If a source was already lossy, converting it to FFV1 does not recover details lost by the earlier encoder.

**Key idea:** Record the pixel format, color interpretation, timestamps, and any preprocessing that happens before a “lossless reference.”

---

# Part II. How a video encoder compresses pictures

## 6. The central idea: remove repeated information

Imagine describing a room in 300 photographs. If the walls do not change, you do not need to describe them from the beginning in every photo. You need to describe what changed. Modern video codecs use a similar principle.

There are two useful kinds of repeated information:

- **Spatial redundancy:** nearby pixels in one image may be similar. A plain wall is a simple example.
- **Temporal redundancy:** much of one frame may look like parts of earlier or later reference frames.

An encoder tries to make a **prediction** and then store only the information needed to correct that prediction. Good predictions leave small correction data, called **residuals**.

```text
Original block:    120 121 120 122
Prediction:        120 120 120 120
Residual:            0   1   0   2
```

An encoder also needs to tell the decoder which prediction method and references to use. Those instructions take bits. So an encoder does not always choose the prediction with the smallest residual. It looks for the best total result.

### 6.1 Entropy: an engineering intuition

**Entropy**, in information theory, is related to how unpredictable symbols are. A long sequence of the same value is often easy to compress. Random, independent values tend to be harder. The exact bit cost depends on context and the coding method. Do not confuse a noisy image with a measurement of real-world “energy”: the words have specific meanings.

### 6.2 Rate–distortion

A codec balances **rate** (bits used) and **distortion** (difference between original and reconstructed content). Reducing bits usually increases some kind of error. The best point depends on the product and the content.

A simple way to think about this is: “Can we save more bits without creating a quality loss that matters?” A more advanced mathematical tool, **rate–distortion optimization (RDO)**, compares encoding choices based on bit cost and a distortion estimate.

**Key idea:** Compression is not only about finding repeated pixels. It is about choosing where bits are worth spending.

## 7. Frames, blocks, and local decisions

A full HD frame has more than two million pixel positions. Modern encoders do not make one single compression decision for the whole image. They divide it into regions called **blocks** and choose prediction and coding methods for these regions.

### 7.1 H.264 macroblocks

H.264 uses a **16 × 16 luma macroblock** as an important coding unit. Inside it, prediction and transforms can use smaller partitions and blocks. The number of macroblocks in a full HD image must allow for right and bottom edges, so use the rounded-up dimensions rather than dividing 1920 × 1080 by 256 without thinking about padding or picture structure.

A macroblock can use intra prediction or inter prediction. Different parts can use different prediction decisions. Smaller partitions can describe detailed motion, but their extra instructions may cost more bits.

### 7.2 HEVC uses CTUs; AV1 uses superblocks

HEVC uses **Coding Tree Units (CTUs)**, which may contain more flexible coding partitions than H.264 macroblocks. AV1 uses **superblocks** and its own recursive partition choices. These are related ideas, but they are **not the same syntax or algorithm**. Never copy a low-level setting across codecs just because the word “block” appears in both.

### 7.3 Coded blocks and skip blocks

A **skip** block is a prediction mode. In H.264, skip uses rules that let some information, such as motion data, be inferred; there is no explicitly coded residual for that skipped macroblock. This can save many bits. It does **not** mean the decoder leaves a hole, discards pixels, or stops rendering that part of the frame.

A high `skip` percentage means prediction works well for many macroblocks. It does not guarantee that total bitrate is low: remaining blocks and other bitstream information may still be costly.

## 8. Intra prediction, inter prediction, and motion

### 8.1 Intra prediction: use neighbors in the same frame

An **intra** block predicts its pixels from already reconstructed neighboring pixels in the same image. For example, a smooth region can often be predicted from its top and left edges. H.264 offers several directional intra prediction modes.

The prediction is not always exact. Residual data describes what is missing. Even an I-frame uses intra prediction, transforms, and often lossy quantization. An I-frame is **not** an uncompressed picture.

### 8.2 Inter prediction: use reference frames

Suppose a ball moves to the right. A useful prediction of the new ball may be a part of a stored reference frame shifted to a new location. The encoder can describe that choice with a **reference index** and a **motion vector**.

```text
Reference frame                    Current frame
+---------------------+            +---------------------+
|  O                  |            |        O            |
|                     |            |                     |
+---------------------+            +---------------------+
      |                                     ^
      +------ reference + movement ---------+
```

Again, prediction is only an estimate. The encoder may send residual data to correct the remaining error.

### 8.3 Motion estimation versus motion compensation

- **Motion estimation:** the encoder searches for a useful reference region and displacement.
- **Motion compensation:** the decoder and encoder build a predicted block from the chosen reference information.

The encoder may try many positions, partitions, and references. More careful searches can improve compression but require more CPU. In x264, `me=hex` selects a hexagonal search method. `subme` controls important refinement steps and related decisions.

### 8.4 Subpixel motion

Objects do not always move by an exact whole number of pixels. H.264 can represent fractional-pixel motion. The decoder uses interpolation rules to build samples between stored pixel positions. This improves prediction in many scenes and adds processing work.

### 8.5 Motion vectors are not a description of real physics

A motion vector says which displacement helps **compression**. It is not a reliable physical measurement of a person's path or a camera's true movement. A codec may choose a useful matching pattern even when no real object moved in that way.

**Key idea:** To estimate compression difficulty, movement alone is not enough. Study how accurately movement can be predicted and how large the residual remains.

## 9. I-frames, P-frames, B-frames, and GOP

### 9.1 I-frame

An **I-picture** is coded without temporal prediction from other pictures. It can still use spatial prediction. It can be compressed and lossy. In a log, an I-frame often needs more bits, but this is not a universal law.

### 9.2 P-frame

A **P-picture** allows inter prediction using a list of references. In a simple teaching example, these references are earlier pictures in display order. Real reference management depends on codec rules. A P-picture can also contain intra-coded regions.

### 9.3 B-frame

A **B-picture** allows two reference lists and may combine predictions. The lists can contain pictures before or after it in display order. In H.264, some B-pictures can also be references for other pictures. B-pictures often improve compression. But they can require frame reordering and buffering.

### 9.4 GOP (Group of Pictures)

A **GOP** is a set of pictures organized around prediction structure and access points. A simple example looks like this:

```text
I  B  B  P  B  B  P  B  B  I
^                           ^
start of one group      next intra picture
```

`keyint=250` usually means a maximum keyframe interval of 250 frames in x264. It does not promise that every keyframe is exactly 250 frames apart. Scene-cut logic may insert a new intra point earlier. At 30 fps, 250 frames are about 8.33 seconds.

### 9.5 I-frame, IDR, and open or closed GOP

An **IDR (Instantaneous Decoder Refresh)** is a special H.264 access point that resets reference dependencies according to H.264 rules. **An I-frame is not always an IDR.** An I-picture can be intra-coded while other pictures still depend on older references.

A **closed GOP** keeps the required decoding dependencies inside its boundary (in the practical packaging sense). An **open GOP** may use references across boundaries. Open GOPs can improve compression, but may complicate independent segment decoding.

An HLS segment of two seconds does not automatically contain everything needed to decode from its first byte. Segment independence requires correct keyframe and reference structure, plus proper packaging and signaling.

### 9.6 Why not make every frame an I-frame?

Inter prediction can save a lot of bits. All-intra encoding may be useful for editing, specialized low latency, or easy random access. But for normal delivery it often needs much more bitrate for similar quality.

### 9.7 GOP and latency

Long GOPs can make seeking and recovery more difficult. B-frame reordering can add buffering delay. These are **different** mechanisms. A long GOP does not always mean high encoding latency, and a short GOP does not remove all latency.

**Key idea:** Choose GOP structure together with your segment duration, playback needs, decoder behavior, and latency budget.

## 10. Residuals, transforms, quantization, and entropy coding

### 10.1 Residual: what prediction could not explain

If the actual sample value is 125 and the predictor gives 120, the residual is +5. Good prediction often produces small residual values. Poor prediction produces larger and more complex residuals. **Residual energy** is an informal way to describe the overall size of these differences; a common technical measure is the sum of squared residual values. You do not need to calculate it by hand to understand its meaning: more remaining difference often takes more bits to describe.

### 10.2 Transform: reorganize the difference

A **transform** converts blocks of residual samples into **coefficients**. Some coefficients represent broad, smooth changes. Others represent fine detail. This often makes the data easier to quantize and compress.

H.264 uses integer transforms with concepts related to DCT, commonly with 4 × 4 or 8 × 8 sizes depending on syntax and mode. `8x8dct` in x264 logs refers to an encoding tool, not a separate video codec.

### 10.3 Quantization: the main deliberate loss

**Quantization** reduces numerical precision. A very simple example:

```text
Before:    3   8   11   19   -4
Rounded:   0  10   10   20    0    (example step = 10)
```

With a bigger step, more values become zero and fewer bits may be needed. But more detail is lost. With a smaller step, more information remains, usually at a higher bit cost.

Real codecs use more complex rules than this simplified example. The essential point remains: **lossy quantization cannot be perfectly reversed**.

### 10.4 Why decoding cannot recover the original

The decoder knows the compressed bitstream and the decoding rules. It does not know the original uncompressed picture. If quantization removed a small coefficient, the decoder cannot guess its exact original value.

### 10.5 Entropy coding: save bits without additional quantization loss

After quantization, many coefficients are zero or small. **Entropy coding** writes these symbols efficiently. H.264 supports **CAVLC** and **CABAC**. CABAC often saves bits but needs more complex processing.

`cabac=1` in an x264 log means CABAC is in use. CABAC itself is not the main cause of lossy quality reduction; the loss mainly comes from earlier choices, especially quantization.

### 10.6 What the decoder does in reverse

The decoder reads coded symbols, reconstructs quantized coefficients, applies inverse transform operations, combines residuals with predictions, and applies required in-loop processing. It reconstructs pictures that can also become references for future predictions. This is why errors in references can influence later pictures.

**Key idea:** Prediction saves repeated information. The transform organizes residual information. Quantization removes precision. Entropy coding stores the remaining symbols efficiently.

## 11. QP, CRF, ABR, CBR, and rate control

This is one of the most important chapters. **QP, CRF, and bitrate are different things.**

### 11.1 QP: how strong is quantization?

The **Quantization Parameter (QP)** helps set how strongly a codec quantizes transform data. For usual 8-bit H.264 use, the basic QP scale is 0–51. Encoder logs may show extended or internal bounds such as `qpmax=69`; those must be read in the context of implementation and bit depth. You do not need to memorize every exception.

In general, a **higher QP** means larger quantization steps, fewer bits, and more distortion, with other conditions unchanged. In H.264, increasing QP by about six steps roughly doubles the quantization step. This does **not** mean visible error doubles or bitrate is cut exactly in half.

QP can change between frames and between regions of one frame. The `Avg QP` reported by x264 is a summary, not proof that every block used one identical QP.

### 11.2 CRF: choose a quality–size policy

**CRF (Constant Rate Factor)** is a rate-control mode used by several software encoders. With x264, you ask for a general quality/bitrate trade-off rather than a fixed output bitrate. The encoder chooses how to spend bits depending on the scene, reference importance, and other decisions.

```bash
ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 18 output.mp4
```

In x264, a lower CRF usually means more bits and less coding distortion. A higher CRF usually means a smaller file and more distortion. **CRF 18 does not mean QP 18 everywhere. It does not mean VMAF 95. It does not promise a particular file size.** CRF scales and behavior are not identical between x264, x265, and SVT-AV1.

### 11.3 Constant QP

```bash
ffmpeg -i input.mp4 -c:v libx264 -qp 22 output_qp.mp4
```

A constant-QP experiment is useful for studying quantization. It is not the same as constant visible quality. Different scenes react differently to the same quantization settings. It also does not produce a constant bitrate.

### 11.4 ABR and VBR

**ABR (Average Bitrate)** aims for a bitrate average. With x264, `-b:v 2500k` requests a target near 2,500 kbit/s. The actual rate changes through time and the overall average may not be exact, especially for short clips.

```bash
ffmpeg -i input.mp4 -c:v libx264 -preset medium -b:v 2500k output_abr.mp4
```

**VBR (Variable Bitrate)** only describes a stream whose rate changes over time. VBR is not one universal rate-control algorithm. CRF and ABR can both produce variable bitrate.

### 11.5 CBR needs a clear definition

**CBR (Constant Bitrate)** is useful when delivery and buffering have strict constraints. But “CBR” may mean a strictly padded transport stream, a bounded rate-control policy, or a nearly constant average over short windows. `-b:v 3000k` by itself is **not** a guarantee of strict CBR. Some encoders can add filler data when configured for it; others do not. Check the actual stream and the requirements of your receiver.

### 11.6 VBV: a model of a bit buffer

**VBV (Video Buffering Verifier)** is a rate/buffer model. Imagine a bucket. Bits arrive at a certain rate. The decoder consumes compressed data according to its schedule. A compliant stream should not break the assumed buffer rules.

```bash
ffmpeg -i input.mp4 -c:v libx264 \
  -b:v 2500k -maxrate 3000k -bufsize 6000k output_vbv.mp4
```

- `-b:v 2500k`: average rate target.
- `-maxrate 3000k`: maximum-rate constraint in the VBV model.
- `-bufsize 6000k`: buffer capacity represented as a number of bits.

A larger modeled buffer allows more variation over time. A smaller one restricts bursts more strongly and may force greater distortion in difficult scenes. `6000 kbit / 3000 kbit/s = 2 s` is a useful **capacity intuition**, not proof that viewer latency is two seconds.

### 11.7 HRD and CPB

**HRD (Hypothetical Reference Decoder)** describes standard-defined timing and buffering behavior. **CPB (Coded Picture Buffer)** is the modeled buffer for compressed picture data. When a log prints CPB fields with zero or `N/A`, this may simply mean certain signaling or information is absent. It does not prove that x264 ignored bitrate allocation.

### 11.8 Two-pass encoding

In **two-pass encoding**, the first pass studies the full program and collects statistics. The second pass uses those statistics to spend a defined bit budget better across easy and difficult scenes. This is often useful for VOD where file size or average bitrate matters. It does not guarantee constant VMAF, and it requires extra time and work.

For live streaming, you cannot run a true full-program first pass before the future content exists.

### 11.9 `qcomp`, `ipratio`, and `pbratio`

These x264 settings influence how rate control distributes quantization across time and picture types. `qcomp` is related to how allocation responds to complexity. `ipratio` and `pbratio` influence quantization relationships between picture types. They are advanced tuning tools, not universal “better quality” switches. Change one carefully, then measure bitrate, quality, and artifacts.

**Key idea:** Rate control decides how to spend a limited resource: **bits**. Quality is measured after encoding. QP and nominal bitrate are not perceptual quality scores.

## 12. Key x264 encoding options

A long x264 log contains many short names. Learn the idea behind each one before changing the default.

### 12.1 `aq`: Adaptive Quantization

**AQ** allows the encoder to allocate bits differently across regions of a frame. A smooth sky and a detailed face may need different treatment. Uniform quantization decisions can waste bits in some places and create visible defects in others.

`aq=1:1.00` means an AQ mode and strength were selected in this x264 build. It does not mean “QP increased by one” or “one percent more bitrate.” AQ algorithms are encoder-specific. When you see banding or loss of local detail, AQ can be one parameter to study, but first make sure the measurement setup is correct.

### 12.2 `mbtree`: Macroblock-tree rate control

**MB-tree** considers a block's value to **future reference pictures**. Imagine a master document that many people will copy. Keeping the master accurate helps all later copies. In a similar way, preserving an important reference region can help many later video frames.

MB-tree uses lookahead information to guide bit allocation. `mbtree=1` means it is enabled. Do not turn it off only because the internal algorithm looks complex. Test the effect against your own content and rate-control mode.

### 12.3 `rc-lookahead`: look into upcoming frames

**Lookahead** studies a window of future frames before final decisions. This can help scene-cut decisions, B-frame placement, and bitrate allocation. It costs memory, processing work, and potentially encoder delay.

With `rc_lookahead=40` at 30 fps, the analysis window represents about 1.33 seconds of picture time. This does **not** mean total end-to-end latency is exactly 1.33 seconds. Internal buffering, scheduling, and the full system also matter.

### 12.4 `b-adapt` and `b-pyramid`

`b-adapt` helps the encoder choose useful B-frame positions. `b-pyramid` allows certain B-frames to become references to other frames within the codec's rules. They can increase compression efficiency, but affect reference dependencies and sometimes compatibility or latency.

### 12.5 `ref`: multiple reference pictures

`ref=3` describes a reference-frame setting in x264. More references can provide better predictions when objects disappear and return, or when motion is complex. They also add memory, search work, and constraints linked to profile or level. “More” is not automatically “better.”

### 12.6 `deblock`: in-loop deblocking

Lossy block coding can produce visible edges between neighboring blocks. The H.264 **in-loop deblocking filter** reduces some of these problems. It affects reconstructed references, not only the final screen display. Changing deblocking can change both perceived sharpness and future prediction.

### 12.7 `psy-rd` and trellis

**Psychovisual optimization** aims to improve what a person notices, rather than only reducing simple pixel error. `psy-rd` is part of x264's approach. **Trellis** helps choose quantized coefficients while considering coding cost. These tools can improve a visual result, but may not always improve a simple metric. This is one reason no single metric should control every engineering decision.

### 12.8 `me`, `subme`, `mixed_ref`, `weightp`, and `weightb`

- **`me`:** motion search method, such as hexagonal search.
- **`subme`:** how much refinement is used for subpixel motion and some mode decisions.
- **`mixed_ref`:** finer reference selection for suitable sub-partitions.
- **`weightp`:** weighted prediction for P-pictures, useful in some brightness changes or fades.
- **`weightb`:** weighted prediction for B-pictures where supported.

These tools may reduce residuals or bit cost, but have CPU and compatibility effects.

### 12.9 `threads`: parallel processing

Multiple threads let a CPU work on several encoding tasks at once. But more threads do not always mean the same speed increase. Video dependencies, scheduling, memory pressure, and encoding settings can limit useful parallelism. On a laptop or under WSL2, also consider the host's other activity.

### 12.10 `tune`: choose an application profile

x264 has tunes such as `film`, `animation`, `grain`, and `zerolatency`. They change groups of options for a purpose. They are **not guarantees**. `zerolatency` cannot remove network, player, or display latency. `grain` does not create a mathematically lossless image.

**Key idea:** Before using an advanced `-x264-params` option, ask what it changes in prediction, allocation, processing cost, or buffering, and how you will measure its benefit.

## 13. Inside H.264 and the decoder

### 13.1 NAL units and the compressed bitstream

H.264 video is organized into **NAL units (Network Abstraction Layer units)**. These hold coded slices, parameter sets, and other information. The structure is more detailed than one packet per frame. A packet shown by FFprobe may not be the same thing as one macroblock, one NAL unit, or one display frame.

Two key metadata structures are:

- **SPS (Sequence Parameter Set):** information about the coded sequence, such as dimensions and decoding properties.
- **PPS (Picture Parameter Set):** information needed to interpret picture coding choices.

**SEI (Supplemental Enhancement Information)** can carry additional information. It does not replace the primary picture data.

### 13.2 `avc1`, `avc3`, and Annex B

H.264 may be stored with different bitstream framing conventions and codec configuration arrangements. MP4 commonly uses length-prefixed NAL units and configuration records; MPEG-TS often uses **Annex B** start-code framing. The labels `avc1` and `avc3` relate to sample entry and parameter-set handling in ISO-based media formats. Changing packaging may need a bitstream filter; it does not necessarily require image re-encoding.

### 13.3 The decoder's job

A conforming decoder does not need to repeat the encoder's expensive search. It reads the instructions that the encoder selected:

1. Read bitstream syntax and entropy-coded symbols.
2. Find prediction type, reference information, and coefficient data.
3. Rebuild prediction blocks and residuals.
4. Apply inverse processes and required in-loop filters.
5. Store reconstructed reference pictures and output pictures in presentation order.

This also explains why the compressed stream is not a simple list of independent JPEG images.

### 13.4 Why encoding is often more expensive

The encoder may examine many candidate vectors, block sizes, modes, quantization decisions, and references. The decoder follows the chosen path. In many workflows this makes software encoding more expensive than decoding, though particular devices and codecs can differ.

### 13.5 Access points and damaged data

If a decoder starts at the wrong point, it may not have the references needed for a picture. Network loss can also damage references and affect later frames. Good delivery design considers recovery, segment independence, correct parameter sets, and error handling.

## 14. Video artifacts: learn to recognize problems

An **artifact** is a visible error or unusual pattern in the output. Knowing how to describe it helps you choose tests and find root causes.

| Artifact | What you may see | A possible technical cause |
|---|---|---|
| Blocking | Square or rectangular patches | Strong quantization and block boundaries |
| Banding | Steps in a smooth gradient | Not enough precision, quantization, or incorrect processing |
| Ringing | Halos around sharp edges | Transform/quantization or resizing filters |
| Blur | Lost fine detail | Quantization, denoising, or scaling |
| Mosquito noise | Moving noise near edges or text | Residual coding and quantization limits |
| Color shift | Wrong or unusual colors | Incorrect matrix, range, primaries, or transforms |
| Stutter / judder | Uneven movement | Frame-rate conversion, timestamps, buffering, or display cadence |
| Tearing | Different parts of one visible image | Display synchronization or rendering |
| Macroblocking after network errors | Broken regions or short bursts of corruption | Damaged/lost reference data or transport |

### 14.1 Not every defect comes from the encoder

Before blaming H.264, check the input, source already encoded with loss, scaling, color conversion, display, network delivery, and player. A good troubleshooting process isolates one stage at a time.

### 14.2 Controlled viewing

When comparing two outputs, use the same time, display, scale, and playback conditions. Check difficult scenes: motion, dark areas, thin lines, faces, grass, water, fine patterns, and gradients. Always write down the timestamp of visible defects.

### 14.3 Banding as a simple experiment

Create a smooth gradient and compress it at different bitrates or bit depths. Compare outputs carefully. This helps you connect numerical precision, quantization, local allocation, and visible bands. **CAMBI** is one metric designed to detect or measure banding changes. It is not a complete picture-quality score.

## 15. Profiles, levels, pixel formats, and compatibility

### 15.1 H.264 Baseline, Main, and High

A **profile** selects groups of codec tools and syntax features. Baseline, Main, and High have different capabilities. High commonly supports useful tools such as 8×8 transforms. Which profile to use depends on target devices and playback requirements.

### 15.2 Codec level

A **level** sets limits on decoding resources, such as frame size in macroblocks, processing rate, reference memory, and coded bitrate constraints. In a log, `level 4.0` is not a quality score. Forcing a level label does not magically make an unsupported stream compatible.

### 15.3 Pixel format matters

`yuv420p` is widely supported by many H.264 players and hardware decoders. But a codec can allow more formats and bit depths in some profiles. Device support is always narrower than the abstract specification. Check 8-bit versus 10-bit, chroma format, color space, and hardware limits.

### 15.4 Hardware decode and real compatibility

A phone or TV may support H.264 but not every profile, level, bit depth, resolution, frame rate, or reference structure. Its decoder may also have practical memory or power constraints. Compatibility is a set of conditions, not one yes/no flag.

**Key idea:** A file that FFplay can decode on a fast computer is not automatically suitable for every device in a commercial streaming service.

---

# Part III. FFmpeg as an engineering tool

## 16. How to read and write FFmpeg commands

FFmpeg is a flexible tool built around **inputs, decoders, filters, encoders, muxers, and outputs**. FFmpeg command-line options are often positional. Where you put an option matters.

```text
Input options  ->  -i input.file  ->  filters / stream selection
             ->  encoder options  ->  output.file
```

### 16.1 A useful command template

```bash
ffmpeg -hide_banner -i input.mp4 \
  -map 0:v:0 -an \
  -vf 'scale=1280:720' \
  -c:v libx264 -preset medium -crf 20 \
  -pix_fmt yuv420p output.mp4
```

Read it from left to right:

- `-hide_banner`: omit the startup banner.
- `-i input.mp4`: open the input.
- `-map 0:v:0`: select the first video stream from the first input.
- `-an`: omit audio in the output.
- `-vf`: apply a video filter, here scaling.
- `-c:v libx264`: encode H.264 using x264.
- `-preset medium`: choose a speed/efficiency setting.
- `-crf 20`: choose a CRF quality–size policy.
- `-pix_fmt yuv420p`: select an output pixel format.
- `output.mp4`: choose the output file and muxer from its extension.

The command may also perform implicit conversions. Inspect logs and output stream information to see what FFmpeg actually selected.

### 16.2 Why `-map` matters

A media file can contain several video tracks, languages, subtitle tracks, attachments, or other streams. Without explicit stream selection, FFmpeg chooses according to its automatic mapping rules. That may be fine for a quick test but unsafe for a production workflow. Use `-map` to say what you want.

### 16.3 Simple and complex filters

`-vf` applies a simple filter chain to a selected video path. `-filter_complex` builds graphs with multiple inputs and outputs. Labels such as `[0:v]` and `[out]` connect graph branches. Use it for comparisons, overlays, multi-input filters, or advanced processing.

### 16.4 Stream copy does not decode video

```bash
ffmpeg -i input.mp4 -map 0 -c copy output.mkv
```

This asks FFmpeg to copy compatible compressed streams into a new container. It does not scale frames, remove video compression artifacts, or change quantization. The output container still has to support those streams.

### 16.5 Logs and error status

FFmpeg writes much diagnostic information to standard error (`stderr`), not standard output. In Bash, `2> encoding.log` saves those messages. Always check the process exit code: a partial output file can exist even when encoding failed.

Useful options include `-loglevel error`, `-loglevel info`, `-report`, `-progress`, and `-benchmark`. Choose verbosity for the problem you are investigating.

### 16.6 WSL2 practice setup

With Ubuntu under WSL2, you can generate and inspect files in a Linux working directory. For heavy tests, keeping temporary work on the WSL Linux filesystem can avoid some cross-filesystem overhead. `ffplay` needs a graphical display: recent Windows 11 WSLg installations provide one, but some setups need display configuration.

## 17. Use FFprobe instead of guessing

### 17.1 Container and stream metadata

```bash
ffprobe -v error -show_format -show_streams -of json input.mp4
```

This can show codec names, dimensions, pixel format, nominal and average frame rates, time bases, duration, audio data, and container metadata. Not every field is always present or reliable. Check the actual stream when something looks wrong.

### 17.2 Frames versus packets

A **packet** is a unit of compressed data passed between demuxers, decoders, encoders, and muxers. A **decoded frame** is an image (or audio data unit) after decoding. Do not assume every codec has exactly one packet for every display frame in every file format or API context.

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries frame=best_effort_timestamp_time,pict_type,key_frame \
  -of csv=p=0 input.mp4 | head
```

`pict_type` tells you the picture type reported by the decoder. `key_frame` tells you its keyframe classification. For H.264 segment independence, investigate IDR/reference structure as well: one field alone may not prove everything you need.

### 17.3 Packets and timestamps

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries packet=pts_time,dts_time,size,flags \
  -of csv=p=0 input.mp4 | head
```

This helps you see decoding versus presentation order and packet sizes. Be careful: packet order is not the same thing as the order in which decoded pictures appear on screen.

### 17.4 Counting real frames

```bash
ffprobe -v error -select_streams v:0 -count_frames \
  -show_entries stream=nb_read_frames,avg_frame_rate \
  -of default=noprint_wrappers=1 input.mp4
```

Counting frames may take time because FFprobe has to read the stream. Metadata fields such as `nb_frames` may be absent or approximate. Verify instead of treating each source number as exact.

### 17.5 The VFR trap

If a screen recording is VFR and you assume equal time between all frames, you can misalign reference and output video. Measure PTS differences. Do not build a quality or synchronization experiment on rounded fps values alone.

**Key idea:** Treat the container's metadata as useful evidence, but verify important claims with packet/frame inspection and actual decoding.

## 18. Filters, resizing, color conversion, and references

### 18.1 Scaling changes the signal

Downscaling from 1080p to 720p removes spatial detail. Upscaling from 720p to 1080p creates interpolated samples; it does not recover the exact lost source detail. Different scaling filters make different trade-offs in sharpness, aliasing, ringing, and speed.

```bash
ffmpeg -i input.mp4 -vf 'scale=1280:720:flags=lanczos' \
  -c:v libx264 -crf 21 scaled.mp4
```

Lanczos may be sharp, but it is not always the best choice. Hard edges can show ringing. Evaluate the scaler as part of the pipeline.

### 18.2 Changing frame rate

`-vf fps=25` produces an output presentation cadence and can drop or duplicate frames. It does not invent correct physical motion between the input frames. Motion interpolation is a separate process and may introduce artifacts.

### 18.3 `format`, `colorspace`, and `zscale`

`format=yuv420p` controls pixel representation. On its own, it does not define a complete and correct change between color spaces or HDR transfer functions. Use color-aware tools such as `colorspace` or `zscale` when required and available. Record color primaries, transfer, matrix, range, and bit depth.

**Important correction:** Changing only metadata cannot repair samples that were converted with the wrong matrix or transfer function.

### 18.4 Generation loss

A lossy video was already changed by quantization. Re-encoding it creates another lossy generation. For a VOD ladder, encode all versions from a high-quality master or mezzanine when possible. Do not build low-bitrate versions by repeatedly re-encoding another low-bitrate rendition.

### 18.5 Seeking and accurate cutting

`-ss` can seek to a time position. Stream-copy cuts often have keyframe and dependency limits. Re-encoding allows more accurate frame cuts but introduces a processing and quality cost. Check the first and last frames, PTS, and actual duration. `-t 6` does not prove every possible file will contain exactly six seconds of displayed pictures.

### 18.6 Keep reference metadata

```yaml
id: testsrc2_720p30_v1
source: testsrc2
frames: 180
fps: 30/1
resolution: 1280x720
pixel_format: yuv420p
reference: reference_720p.mkv
source_codec: ffv1
transformations_before_reference: none
```

Your experiment result should be traceable to a specific reference and command. One isolated CRF value is not enough to reproduce a test weeks later.

## 19. Encoder presets, CPU cost, and fair benchmarks

### 19.1 A preset is not a quality score

x264 presets range from `ultrafast` through `medium`, `slow`, and more demanding settings. A slower preset usually spends more CPU searching for efficient coding choices. At a fixed bitrate it may give better quality. At a fixed CRF it can change both bitrate and quality. Do **not** say that a preset changes only speed.

### 19.2 What `fps`, `speed`, and `elapsed` really measure

FFmpeg may measure an entire workflow: demux, decode, filter, encode, mux, and I/O. `speed=4.7x` means the job processes media at about 4.7 times playback speed in that run. It does not necessarily measure only the encoder's internal speed.

For fair tests, keep inputs, filters, hardware, threads, and output behavior stable. Repeat tests and report typical values, such as the median. Very short clips may be dominated by process startup or I/O overhead.

### 19.3 Minimum operational metrics

Record actual encode time, output size, average bitrate, bitrate over short windows, CPU and memory use, error counts, reference and B-frame behavior, GOP/keyframe positions, compatibility results, and quality metrics where available.

### 19.4 Why “we saved 5%” is not enough

A five-percent bitrate saving is not automatically good if encoding cost becomes ten times higher, difficult scenes become worse, or a target device cannot decode the result. Evaluate the full product trade-off.

### 19.5 Two different benchmark questions

- **Equal bitrate:** Which encoder or preset delivers better quality for the same bit budget?
- **Equal CRF:** How does the encoder's own quality–size policy respond to a different preset?

Do not mix these into one conclusion. Report which conditions were fixed and which were measured.

## 20. H.264, HEVC, and AV1: shared ideas and key differences

### 20.1 What they share

All three use spatial and temporal prediction, transforms, quantization, entropy coding, and rate–distortion decisions. Learning H.264 gives you a strong foundation for the others. But **CRF values, QP values, presets, profiles, and levels are not directly interchangeable between codecs**.

| Topic | H.264 / AVC | HEVC / H.265 | AV1 |
|---|---|---|---|
| Common FFmpeg software encoder | `libx264` | `libx265` | `libsvtav1` or `libaom-av1` |
| Main block structure | 16×16 macroblocks with partitions | CTUs with flexible partitions | Superblocks with flexible partitions |
| Temporal prediction | Yes | Yes, with more tools | Yes, with advanced options |
| Common strength | Very broad compatibility | Good efficiency in many uses | Good efficiency with modern tool set |
| Encoding CPU cost | Wide range | Often higher for similar goals | Highly dependent on encoder and preset |
| CRF 25 compared with another codec | Not equivalent | Not equivalent | Not equivalent |

No codec wins in all conditions. Content, bitrate, resolution, implementation, quality target, and hardware support matter.

### 20.2 HEVC / H.265

HEVC supports larger coding-tree structures and more flexible prediction and partition choices. It is used for many high-resolution, UHD, and HDR workflows. `libx265` has its own lookahead, presets, and rate control. Check both processing cost and playback support. Commercial deployment may also involve licensing and patent considerations.

```bash
ffmpeg -i input.mp4 -map 0:v:0 -an \
  -c:v libx265 -preset medium -crf 28 -pix_fmt yuv420p \
  output_hevc.mp4
```

`CRF 28` is an example starting point, not a quality equivalent to `CRF 28` in x264.

### 20.3 AV1

AV1 has advanced prediction and transform options, plus other coding tools. On FFmpeg, `libsvtav1` is one useful software encoder when the build includes it.

```bash
ffmpeg -i input.mp4 -map 0:v:0 -an \
  -c:v libsvtav1 -preset 8 -crf 35 -pix_fmt yuv420p \
  output_av1.mkv
```

SVT-AV1 preset and CRF values belong to that encoder. Use `ffmpeg -h encoder=libsvtav1` to inspect your installed version. Before a real service deployment, test clients, hardware decode, power use, and container support.

### 20.4 A fair comparison

1. Use the **same source master** and time interval.
2. Keep resolution, frame rate, and color handling consistent.
3. Decide whether to compare at **equal bitrate**, **equal quality**, or **equal encode time**.
4. Measure several bitrate points rather than one file.
5. Use aligned objective metrics and visual checks.
6. Record codec, encoder version, preset, hardware, and output settings.
7. Check playback and decode cost, not only compression efficiency.

### 20.5 Avoid copying codec-specific settings

HEVC QP is not the same as H.264 QP for quality comparison. SVT-AV1 has different internal tools from x264. A multi-codec workflow should use common external data (source, bitrate, resolution, measured quality) and separate codec-specific settings.

---

# Part IV. Quality, transport, and streaming

## 21. How to measure video quality

Compression is not only about reducing bytes. We need to keep enough visible information. A good evaluation separates **pixel error**, **estimated perceptual quality**, and **real playback experience**.

### 21.1 Reference and distorted video

A **reference** is the source against which we compare an encoded output. A **distorted** video is the output under test. They must show the **same picture at the same time** and use consistent resolution, frame rate, pixel representation, and color interpretation. A one-frame alignment error can make a metric look very bad even when the codec works well.

### 21.2 MSE, PSNR, and SSIM

- **MSE (Mean Squared Error):** average squared difference between corresponding sample values. Large errors receive more weight.
- **PSNR (Peak Signal-to-Noise Ratio):** a logarithmic measure in decibels (dB). Higher usually means lower sample error. Identical compared samples give infinite PSNR.
- **SSIM (Structural Similarity):** compares aspects of local structure, brightness, and contrast. It may agree better with human perception than MSE under some kinds of distortion.

These metrics do not understand a story or its meaning. Small changes in timing, scaling, or brightness can greatly affect scores. A higher score is only meaningful under a defined and fair test.

### 21.3 VMAF: a perceptual model

**VMAF (Video Multi-Method Assessment Fusion)** combines visual features using a model trained to relate measurements to human quality ratings in particular viewing conditions. It is a **full-reference** method: it needs both the reference and the distorted image.

The **model name, version, and evaluation conditions matter**. Different VMAF model families or updates may give different values. Check the installed model and official documentation rather than assuming a default. Models evolve; the numerical score is not an absolute physical measurement.

**VMAF 95 is not “95 percent quality.”** It does not guarantee that no person will see a defect. A threshold can be useful as an engineering target only after you validate it for the content, model, viewing conditions, and product.

### 21.4 VMAF when resolution changes

Suppose the source is 1080p and the coded output is 720p. You must decide the evaluation resolution. If a 720p result will be upscaled on a 1080p display, a comparison after controlled upscaling can be relevant. Another study may compare native-size material. These answer different questions.

Document which scaler, pixel format, and resolution were used. Do not compare two encoders after different accidental preprocessing.

### 21.5 CAMBI and banding

**CAMBI (Contrast Aware Multiscale Banding Index)** measures visible banding or contouring. In normal use, a **higher CAMBI** means more banding. It can provide useful information that VMAF alone may miss. Depending on software and options, CAMBI can use reference-related processing or run without a reference.

### 21.6 Scores over time: pooling

One video may have an average VMAF of 96 while still containing some very bad scenes. Therefore, look at:

- **Average:** the overall result under the chosen aggregation.
- **Low percentiles:** how poor the more difficult frames become.
- **Worst scene or segment:** whether one part violates a required quality level.
- **Quality versus bitrate:** whether more bits still bring enough benefit.

The average of five scene averages is **not** always the same as an average over every frame, because the scenes may have different lengths. Say whether you use equal-scene weighting or frame/duration weighting.

### 21.7 Objective metrics and human review

Look at the actual pictures. Perform small blind visual tests when an important product decision depends on quality. Sharpening and denoising can influence numerical scores in ways that do not match source fidelity. VMAF has also supported approaches such as NEG to reduce certain metric-gaming problems. The right method depends on the question and the exact model.

**Key idea:** Metrics help you compare under a controlled method. They do not replace viewing tests or knowledge of the player and display.

## 22. Rate–distortion experiments and smart bitrate selection

### 22.1 The bitrate–quality curve

Encode the same reference several times at increasing target rates, perhaps 500, 1,000, 2,000, 3,000, and 5,000 kbit/s. For each output, record actual bitrate and VMAF or another quality measure.

```text
Estimated quality
100 |                          .___
 95 |                    .____/
 90 |              .____/
 80 |        .____/
    +----------------------------------> Bitrate
        less                         more
```

This is a teaching diagram. Real curves may be less smooth. Often the first bitrate increases help a lot; later increases produce only small quality gains. This is called **diminishing returns**.

### 22.2 Finding the smallest bitrate for a quality target

For a fixed scene and configuration, you may search for the lowest bitrate that meets a quality rule, such as a validated VMAF target. **Binary search** can be useful when quality increases sufficiently smoothly with bitrate. Real measurements can have small non-monotonic changes because of mode selection, GOP, and sampling. Save every tested point and use a safe fallback when an assumption fails.

### 22.3 Content-aware encoding

Different scenes need different bitrates to reach similar perceptual quality. A still room is often easy to predict. Moving water, grass, grain, and complex reflections can be harder. A content-aware system may use motion, texture, edges, noise, prediction error, and earlier measurements to estimate a useful rate.

**Important:** Do not train a model only on simple synthetic patterns and assume it will work on real films or sports. Test it on content outside the training dataset.

### 22.4 Convex hull and the encoding ladder

Suppose you have many candidates with different bitrates and resolutions. Some are **dominated**: another candidate gives higher quality at a lower bitrate under the same evaluation method and compatibility rules. Dominated points are often poor ladder choices.

The **rate–distortion frontier** contains useful efficient candidates. In mathematical optimization, a convex hull has a more specific geometric meaning; practical ladder design also needs to account for device support, bitrate peaks, switching, and playback requirements.

### 22.5 Designing a controlled experiment

Write down:

1. **Hypothesis:** for example, “A shorter GOP improves random access but may increase bitrate.”
2. **Independent variable:** the parameter being changed, such as GOP length.
3. **Controlled variables:** source, codec implementation, preset, FPS, color conversion, hardware conditions.
4. **Measured results:** bitrate, file size, VMAF, PSNR, CPU time, latency, keyframe positions.
5. **Acceptance rule:** what must improve, and what may not become worse.

Change one variable at a time while learning. Later, controlled multi-variable experiments can answer more advanced product questions.

### 22.6 Do not confuse correlation with cause

If you compare H.264 at 1080p and 3 Mbps with AV1 at 720p and 2 Mbps, many things changed at once. It can be a useful **product configuration comparison**, but it cannot tell you how much of the difference came from the codec alone.

### 22.7 Keep negative results

A good experiment may prove that a tuning idea does not help. Save those results. Clear evidence that something fails can prevent repeated work and wrong assumptions later.

## 23. Audio, synchronization, and muxing

Even a video-focused engineer must understand enough audio to avoid damaging playback.

### 23.1 Samples, sample rate, channels, and audio codecs

Audio is sampled through time. **48 kHz** means 48,000 samples per second **per channel**. Stereo has two channels. A typical 5.1 layout uses six channels with different roles. Common codecs include AAC, Opus, AC-3, and E-AC-3. Choose based on player support, bitrate, latency, quality, and licensing requirements.

### 23.2 Audio and video have different clocks

Video has presentation times for pictures. Audio has presentation timing for groups of sound samples. If the clocks or timestamp conversion are wrong, spoken words and mouth movement may drift apart. A fixed offset and a slowly increasing drift are different problems.

### 23.3 Use timestamps to diagnose sync

When remuxing, trimming, or changing frame rate, compare the first and last timestamps of both audio and video. Inspect missing timestamps, sudden jumps, and discontinuities. Do not hide the symptom by adding global timestamp options without understanding the source timing.

### 23.4 Loudness is part of quality

A good video picture can still be a poor experience if audio is too quiet, clips, or changes level between programs. Basic terms to know include **peak**, **true peak**, **integrated loudness (LUFS)**, and loudness normalization. Exact targets depend on delivery standards and platforms.

### 23.5 Audio in our labs

FFmpeg can generate audio with `sine`. We can join this with a synthetic video signal to inspect two streams and check timing. This helps validate muxing and synchronization without licensed media.

## 24. From an MP4 file to HLS, DASH, and a CDN

### 24.1 One file versus adaptive delivery

A single MP4 is useful for testing, download, and progressive playback. Many OTT services instead provide **ABR (Adaptive Bitrate) streaming**. The player chooses among several versions of the same program. The versions differ in bitrate, resolution, sometimes codec, and other properties.

```text
High-quality master
  +-- 1080p / 5 Mbps ---- segments --+
  +--  720p / 3 Mbps ---- segments --+--> playlist / manifest --> ABR player
  +--  540p / 1.5 Mbps -- segments --+
  +--  360p / 0.8 Mbps -- segments --+
```

The set of alternatives is an **encoding ladder**. A **manifest** or playlist tells the player where the media is and what alternatives exist. The player chooses based on its buffer, estimated network capacity, display needs, and other policies.

### 24.2 HLS and MPEG-DASH

**HLS (HTTP Live Streaming)** uses M3U8 playlists and media segments, such as MPEG-TS or fragmented MP4, depending on the selected delivery profile. **MPEG-DASH** uses an MPD description and media segments. Both separate information **about the presentation** from the actual **media data**.

### 24.3 A scene, GOP, and segment are different

- A **scene** is a visual or semantic unit, such as one camera shot.
- A **GOP** is an encoding/reference structure between pictures and access points.
- A **segment** is a unit of media delivery.

One scene may cross several segments. A segment may contain several GOPs. They can be aligned for a specific design, but they are not identical concepts.

### 24.4 Why alignment matters for ABR

The player may switch between representations. To do this safely, the alternatives need matching timeline positions and suitable independent access points. Poor segment/keyframe alignment can cause stalls, glitches, or failed switches. Test the actual player behavior, not just the playlist syntax.

### 24.5 IDR and segment creation

An independently decodable H.264 segment needs the required reference and parameter information at the access point. Many HLS authoring recommendations use regular IDRs and aligned segment boundaries. A **2-second GOP** is a common example, not a universal rule for all streaming. Some HLS guidelines and profiles recommend other segment sizes, including around six seconds for conventional delivery.

### 24.6 Origin, CDN, and cost

The **origin** stores or serves media. A **CDN (Content Delivery Network)** caches content nearer to users. Cost depends on delivered bytes, concurrent viewers, cache hits, request patterns, and platform design. Saving bits matters most where it affects content users actually watch.

### 24.7 The client chooses ABR quality

The server supplies alternatives. The **player** usually decides which one to fetch next. A correct encoding ladder cannot guarantee smooth playback if the network varies, bitrate peaks are too high, or the player makes poor decisions.

### 24.8 Buffer and bandwidth

A six-second video segment averaging 3 Mbps contains about 18 Mbit of **video payload**, before other overhead and audio. If the effective network delivers only 2 Mbps while media use remains near 3 Mbps, the player will spend its buffer and eventually stall unless it adapts or has enough reserve.

**Key idea:** A streaming experience depends on codec, GOP, segments, manifest, CDN, player, network, and device **together**.

## 25. Live streaming and low latency

### 25.1 What does latency mean?

**End-to-end latency** is the time from an event happening in front of a camera to a viewer seeing it. It is not only the time spent in `libx264`.

```text
Capture -> preprocessing -> encoder -> ingest/origin -> packaging
        -> CDN/network -> player buffer -> decoder -> display
```

Every stage may add delay. Low latency requires work on the full chain, including variability and recovery after problems. `-tune zerolatency` cannot remove all the delay outside the encoder.

### 25.2 A live encoder cannot know the full future

For VOD, the complete program already exists. An encoder can analyze future frames, plan cuts, and allocate bits over the whole video. A live encoder has only a limited future window. Waiting for ten seconds of future pictures is not acceptable if your product requires very low delay.

Lookahead and B-frame reordering therefore involve stronger compromises in live systems.

### 25.3 Bitrate control in live video

Live streaming often needs a limit on bursts to protect network and buffering. VBV is useful here. However, tight limits can reduce image quality in hard scenes. Engineers must consider average rate, maximum rate, buffer model, network conditions, and player behavior.

**Bitrate efficiency still matters in live streaming.** The main difference from VOD is the tighter time and resource budget.

### 25.4 Standard HLS, LL-HLS, and other protocols

Conventional full-segment HLS can add noticeable delay. **Low-Latency HLS (LL-HLS)** reduces some of that delay by using smaller segment parts and a delivery model designed to make media available earlier. Other systems include RTP, WebRTC, and protocols such as SRT for different contribution or distribution uses. They have different latency, reliability, scale, and infrastructure trade-offs.

### 25.5 Monitor the actual service

Track input and output frame timing, PTS/DTS, encode speed, dropped frames, segment arrival times, player buffer, origin/CDN errors, restarts, measured viewer latency, bitrate by time window, audio/video synchronization, and sampled visual quality. A process may show “healthy” while real viewers still suffer.

### 25.6 VMAF for live monitoring

VMAF needs aligned reference and output images. In live systems you can keep recent windows and evaluate them with a delay. This can support monitoring and later tuning. But it does **not** give a zero-delay, no-cost quality answer for every incoming frame. Fast signals and encoder feedback are often needed for immediate decisions.

### 25.7 A practical first design

Set a realistic encoder processing budget. Choose rate-control settings that work with delivery constraints. Align GOPs and independent access points for packaging. Verify end-to-end latency and buffering. Use low-cost measurements during live operation, and deeper perceptual checks afterward.

**Key idea:** VOD can use knowledge of the whole program. Live video must decide with limited knowledge of the future. This changes the encoder and delivery design.


---

# Part V. Guided FFmpeg labs for Ubuntu on WSL2

These labs use generated video and do not need copyrighted media. Most sources are only **640×360, 30 fps, and six seconds** to save CPU and disk space. Repeat with 1080p after you understand each mechanism. Your exact bitrate, QP, and encoding speed may differ from the example because of FFmpeg version, x264 version, CPU, and settings.

## Prepare your lab

```bash
sudo apt update
sudo apt install -y ffmpeg jq python3 time
mkdir -p ~/video-lab
cd ~/video-lab
ffmpeg -hide_banner -version | head -n 3
ffmpeg -hide_banner -encoders | grep -E 'libx264|libx265|libsvtav1|ffv1'
ffmpeg -hide_banner -filters | grep -E 'testsrc2|libvmaf|psnr|ssim'
```

**Safety:** Work only inside `~/video-lab`. The `-y` option overwrites output files. Save command lines and program versions with every result. For multiline Bash commands, `\` must be the final character of the line. `libvmaf` is optional; you can complete the other labs without it. Keep the same reference for every quality comparison.

## Lab 1. Create and play your first video

**Goal:** Learn to separate the synthetic source, the encoder, the container, and the player. The input is generated locally; you do not need a camera or a downloaded film.

The first command creates six seconds of moving test pictures. The second starts a player (a graphical display is needed in WSL).

```bash
ffmpeg -y -hide_banner \
  -f lavfi -i 'testsrc2=size=640x360:rate=30:duration=6' \
  -c:v libx264 -preset medium -crf 18 -pix_fmt yuv420p \
  l01_test.mp4

ffplay -autoexit l01_test.mp4
```

Inspect the output instead of assuming its properties:

```bash
ffprobe -v error -show_entries \
  format=duration,size,bit_rate:stream=codec_name,width,height,pix_fmt,avg_frame_rate \
  -of json l01_test.mp4
```

**What to learn:** Expected: one H.264 video stream, 640×360, near six seconds, and about 30 fps. Its size depends on the pattern and the encoder. **Think:** What changes if you remux to MKV without re-encoding? Which step creates the pixels shown on screen?

## Lab 2. Create a lossless reference and check it

**Goal:** Learn why a reference should be clearly defined before a lossy experiment. The source is generated and stored with FFV1.

Make a lossless reference and one deliberately lossy result:

```bash
ffmpeg -y -hide_banner \
  -f lavfi -i 'testsrc2=size=640x360:rate=30:duration=6' \
  -pix_fmt yuv420p -c:v ffv1 l02_ref.mkv

ffmpeg -y -hide_banner -i l02_ref.mkv \
  -c:v libx264 -preset medium -crf 28 l02_crf28.mp4
```

The next commands write a frame-hash list, encode the reference again with FFV1, and compare decoded frame hashes:

```bash
ffmpeg -v error -i l02_ref.mkv -f framemd5 l02_ref.framemd5
ffmpeg -v error -i l02_ref.mkv -c:v ffv1 l02_copy.mkv
ffmpeg -v error -i l02_copy.mkv -f framemd5 l02_copy.framemd5

diff -u l02_ref.framemd5 l02_copy.framemd5
```

**What to learn:** The decoded sample hashes should match when frames, sample format, and timing remain unchanged. The full `framemd5` files also contain headers; if `diff` finds a difference, inspect whether it is a header change or a frame-hash change. **Think:** Why does FFV1 not restore RGB chroma that you already removed by converting to 4:2:0?

## Lab 3. File size and bitrate

**Goal:** Connect bits per second, video duration, and bytes on disk.

Measure the duration, size, and reported bitrate, then compare the lossless and lossy files:

```bash
ffprobe -v error -show_entries format=duration,size,bit_rate \
  -of default=noprint_wrappers=1 l02_crf28.mp4

stat -c '%n %s bytes' l02_ref.mkv l02_crf28.mp4
```

**What to learn:** Estimate size using `megabytes ≈ Mbps × seconds / 8`. For example, 3 Mbps for one minute is roughly 22.5 MB of video payload, before audio and container overhead. **Practice:** Compare ten hours at 2 Mbps and ten hours at 5 Mbps. Why does saved storage not prove that picture quality is acceptable?

## Lab 4. Static versus changing video

**Goal:** Observe why two videos with the same resolution, fps, duration, and CRF can have very different file sizes.

Create a still gray source and a moving `testsrc2` source with the same H.264 settings:

```bash
ffmpeg -y -hide_banner -f lavfi \
  -i 'color=c=gray:size=640x360:rate=30:duration=6' \
  -c:v libx264 -preset medium -crf 18 -pix_fmt yuv420p l04_static.mp4

ffmpeg -y -hide_banner -f lavfi \
  -i 'testsrc2=size=640x360:rate=30:duration=6' \
  -c:v libx264 -preset medium -crf 18 -pix_fmt yuv420p l04_dynamic.mp4

ffprobe -v error -show_entries format=duration,size,bit_rate \
  -of default=noprint_wrappers=1 l04_static.mp4
ffprobe -v error -show_entries format=duration,size,bit_rate \
  -of default=noprint_wrappers=1 l04_dynamic.mp4
```

**What to learn:** A still source usually compresses much better because most later frames are easy to predict. Exact sizes depend on the implementation. **Explain:** Why is “same CRF means same file size” wrong? How would random noise change this test?

## Lab 5. Time precision at 25 fps and 30 fps

**Goal:** Learn how frame rate affects frame counts and time positions.

Make two lossless six-second sources and count their decoded frames:

```bash
ffmpeg -y -hide_banner -f lavfi \
  -i 'testsrc2=size=640x360:rate=25:duration=6' \
  -c:v ffv1 l05_25fps.mkv

ffmpeg -y -hide_banner -f lavfi \
  -i 'testsrc2=size=640x360:rate=30:duration=6' \
  -c:v ffv1 l05_30fps.mkv

for f in l05_25fps.mkv l05_30fps.mkv; do
  echo "### $f"
  ffprobe -v error -select_streams v:0 -count_frames \
    -show_entries stream=avg_frame_rate,nb_read_frames \
    -of default=noprint_wrappers=1 "$f"
done
```

**What to learn:** Expected under these constant-rate settings: about 150 frames at 25 fps and 180 at 30 fps. The last frame begins before the end of the six-second interval, so an FFmpeg progress time may be shorter than exactly `00:00:06.00`. **Think:** Why would it be unfair to blame the encoder for a bitrate change when fps also changed?

## Lab 6. Controlled movement with `overlay`

**Goal:** Build an easy-to-understand moving object. This is useful for studying prediction and motion vectors.

The overlay position `40+60*t` moves an 80×80 white square by 60 pixels each second:

```bash
ffmpeg -y -hide_banner \
  -f lavfi -i 'color=c=black:s=640x360:r=30:d=6' \
  -f lavfi -i 'color=c=white:s=80x80:r=30:d=6' \
  -filter_complex "[0:v][1:v]overlay=x='40+60*t':y=140:shortest=1" \
  -c:v ffv1 l06_motion_ref.mkv

ffplay -autoexit l06_motion_ref.mkv
```

Now encode the generated FFV1 reference with H.264:

```bash
ffmpeg -y -i l06_motion_ref.mkv -c:v libx264 \
  -crf 22 -preset medium l06_motion.mp4
```

**What to learn:** The black background is easy to predict; moving square edges need more decisions. A real encoder may not select a vector that exactly matches the visible object movement. Repeat with `40+30*t` and `40+90*t`. Compare size, skip percentages, and quality against the **corresponding** reference for each variation.

## Lab 7. Inspect I, P, B, and keyframe positions

**Goal:** Look at the real picture structure instead of trusting a simple GOP diagram.

Encode with a regular 60-frame GOP configuration and inspect picture metadata:

```bash
ffmpeg -y -i l02_ref.mkv -c:v libx264 \
  -preset medium -crf 20 -g 60 -keyint_min 60 \
  -sc_threshold 0 -x264-params open-gop=0 \
  l07_gop60.mp4

ffprobe -v error -select_streams v:0 \
  -show_entries frame=best_effort_timestamp_time,pict_type,key_frame,pkt_size \
  -of csv=p=0 l07_gop60.mp4 | head -n 35
```

Now inspect compressed packets and compare their timestamps:

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries packet=pts_time,dts_time,size,flags \
  -of csv=p=0 l07_gop60.mp4 | head -n 15
```

**What to learn:** At 30 fps, a 60-frame interval corresponds to two seconds. Check actual results. `pict_type=I` alone does not prove that all segment dependencies are independent. **Think:** What is the difference between decoded-frame presentation order and packet decoding order?

## Lab 8. Change GOP length and measure cost

**Goal:** Compare frequent access points against longer groups.

Make one output with a 30-frame GOP and another with 180 frames, keeping the other main settings the same:

```bash
ffmpeg -y -i l02_ref.mkv -c:v libx264 -preset medium -crf 20 \
  -g 30 -keyint_min 30 -sc_threshold 0 -x264-params open-gop=0 \
  l08_gop30.mp4

ffmpeg -y -i l02_ref.mkv -c:v libx264 -preset medium -crf 20 \
  -g 180 -keyint_min 180 -sc_threshold 0 -x264-params open-gop=0 \
  l08_gop180.mp4

stat -c '%n %s bytes' l08_gop30.mp4 l08_gop180.mp4
```

**What to learn:** Longer GOPs can reduce the cost of frequent intra pictures, but content and scene changes affect the outcome. At fixed CRF this is **not** an equal-bitrate comparison. For a fair efficiency study, also compare at equal bitrate or measured quality. **Think:** How would a six-second GOP affect seeking or two-second ABR segments?

## Lab 9. Read average QP and I/P/B counts

**Goal:** Connect the final x264 summary to what you learned about picture types and rate control.

Save the encoder log and filter for important summary lines:

```bash
ffmpeg -y -hide_banner -loglevel info \
  -i l02_ref.mkv -c:v libx264 -preset medium -crf 18 \
  l09_crf18.mp4 2> l09_encode.log

grep -E 'frame I:|frame P:|frame B:|kb/s:|mb P|mb B|profile|rc=crf' \
  l09_encode.log
```

The summary usually looks like this:

```text
frame I:N  Avg QP:... size:...
frame P:N  Avg QP:... size:...
frame B:N  Avg QP:... size:...
```

**What to learn:** A larger Avg QP often means stronger quantization, but I, P, and B pictures are structurally different. **Write an answer:** Why can a B-picture use a higher QP while still looking reasonable? Discuss reference prediction, not only QP.

## Lab 10. Compare CRF, constant QP, and target bitrate

**Goal:** Separate three different questions: intended quality–size policy, quantization, and target average bitrate.

Encode the same reference using three rate-control modes and inspect the sizes:

```bash
ffmpeg -y -i l02_ref.mkv -c:v libx264 -preset medium -crf 18 l10_crf.mp4
ffmpeg -y -i l02_ref.mkv -c:v libx264 -preset medium -qp 22 l10_qp.mp4
ffmpeg -y -i l02_ref.mkv -c:v libx264 -preset medium -b:v 800k l10_abr.mp4

for f in l10_*.mp4; do
  echo "$f"
  ffprobe -v error -show_entries format=duration,size,bit_rate \
    -of default=noprint_wrappers=1 "$f"
done
```

**What to learn:** Do **not** compare the numbers `CRF 18` and `QP 22` as if they used the same scale. ABR does not promise exactly 800,000 bits in every second. Build a table of size, rate, I/P/B average QP, and visible defects. Which command asks for an average rate? Which asks for a quality–size policy?

## Lab 11. ABR versus VBV limits

**Goal:** Learn why an average bitrate goal does not control every short-term burst.

Encode once with a bitrate target and once with VBV-related limits:

```bash
ffmpeg -y -i l02_ref.mkv -c:v libx264 -preset medium \
  -b:v 1000k l11_abr.mp4

ffmpeg -y -i l02_ref.mkv -c:v libx264 -preset medium \
  -b:v 1000k -maxrate 1100k -bufsize 2200k l11_vbv.mp4
```

Then inspect packet sizes and times:

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries packet=pts_time,size,flags \
  -of csv=p=0 l11_vbv.mp4 | head -n 20
```

**What to learn:** Repeat with a smaller `-bufsize`, such as `500k`, and inspect changes. A six-second clip may be too short for clean average-rate comparisons. **Think:** Why might playback stall even if average video bitrate is lower than average network throughput? Include audio, short bursts, requests, and player buffering.

## Lab 12. Two-pass encoding for VOD

**Goal:** Learn why seeing the entire program first can improve spending a fixed number of bits.

Run first pass and second pass using the same input and target:

```bash
ffmpeg -y -i l02_ref.mkv -map 0:v:0 -an \
  -c:v libx264 -preset slow -b:v 900k \
  -pass 1 -passlogfile l12_stats \
  -f null /dev/null

ffmpeg -y -i l02_ref.mkv -map 0:v:0 -an \
  -c:v libx264 -preset slow -b:v 900k \
  -pass 2 -passlogfile l12_stats \
  l12_2pass.mkv
```

Compare the result with a one-pass encode:

```bash
ffmpeg -y -i l02_ref.mkv -map 0:v:0 -an \
  -c:v libx264 -preset slow -b:v 900k \
  l12_1pass.mkv
```

**What to learn:** Record actual size, bitrate, encode time, and quality. Short, simple synthetic clips may show only small differences. For a serious benchmark, also inspect first-pass settings and use longer varied content. **Think:** Why can a live event not use a true whole-program first pass?

## Lab 13. Compare x264 presets

**Goal:** Measure the speed/efficiency trade-off with a fixed bitrate target.

Run `veryfast`, `medium`, and `slow` on the same source:

```bash
for preset in veryfast medium slow; do
  /usr/bin/time -f "${preset}: %e s reales" \
    ffmpeg -y -hide_banner -loglevel warning -i l02_ref.mkv \
      -c:v libx264 -preset "$preset" -b:v 900k \
      "l13_${preset}.mp4"
done
```

**What to learn:** Install package `time` if `/usr/bin/time` is missing. Run longer samples and repeat tests before making performance claims. Compare real elapsed time and quality at the target bitrate. A slower preset does not guarantee a visible gain for a simple synthetic source.

## Lab 14. Aligned PSNR and SSIM measurements

**Goal:** Practice quality checks that compare the same original and coded frames.

Run the two full-reference filters with explicit start-time alignment:

```bash
ffmpeg -hide_banner -i l10_crf.mp4 -i l02_ref.mkv \
  -filter_complex '[0:v]setpts=PTS-STARTPTS[dist];[1:v]setpts=PTS-STARTPTS[ref];[dist][ref]psnr=stats_file=l14_psnr.log' \
  -f null -

ffmpeg -hide_banner -i l10_crf.mp4 -i l02_ref.mkv \
  -filter_complex '[0:v]setpts=PTS-STARTPTS[dist];[1:v]setpts=PTS-STARTPTS[ref];[dist][ref]ssim=stats_file=l14_ssim.log' \
  -f null -
```

**What to learn:** A higher PSNR generally means less sample error; SSIM describes a different kind of similarity. Neither is a universal viewing-quality number. Compare both `l02_crf28.mp4` and `l10_crf.mp4` against **the same** `l02_ref.mkv` reference. If the scores are surprising, check alignment, frame rate, color, and format negotiation.

## Lab 15. Measure VMAF when available

**Goal:** Learn to check optional FFmpeg features and to record the metric model.

Check whether this FFmpeg build contains `libvmaf` and what syntax it accepts:

```bash
ffmpeg -hide_banner -filters | grep libvmaf
ffmpeg -hide_banner -h filter=libvmaf
```

If supported, run a basic full-reference comparison:

```bash
ffmpeg -hide_banner -i l10_crf.mp4 -i l02_ref.mkv \
  -filter_complex '[0:v]setpts=PTS-STARTPTS[dist];[1:v]setpts=PTS-STARTPTS[ref];[dist][ref]libvmaf=log_fmt=json:log_path=l15_vmaf.json' \
  -f null -
```

**What to learn:** Read the JSON: distinguish per-frame scores from pooled results. Keep track of the model name, version, input order, resizing, and format handling. A VMAF score from one model may not be comparable to another. **If libvmaf is absent**, use PSNR and SSIM but clearly state that they are different metrics.

## Lab 16. Encode HEVC and AV1 candidates

**Goal:** Learn to separate codec standards from encoder-specific options and numeric scales.

Check that your FFmpeg includes suitable software encoders:

```bash
ffmpeg -hide_banner -encoders | grep -E 'libx265|libsvtav1'
```

Make one HEVC candidate and one AV1 candidate:

```bash
ffmpeg -y -i l02_ref.mkv -an \
  -c:v libx265 -preset medium -crf 28 -pix_fmt yuv420p \
  l16_hevc.mkv

ffmpeg -y -i l02_ref.mkv -an \
  -c:v libsvtav1 -preset 8 -crf 35 -pix_fmt yuv420p \
  l16_av1.mkv
```

Inspect the resulting streams:

```bash
for f in l16_hevc.mkv l16_av1.mkv; do
  echo "--- $f"
  ffprobe -v error -show_entries stream=codec_name,profile,pix_fmt,width,height:format=size,duration \
    -of default=noprint_wrappers=1 "$f"
done
```

**What to learn:** The example CRF values are not quality-equivalent. Do not claim that one codec wins from this pair. For a fair study, encode several rates, use the same reference, and compare quality, encode time, playback support, and decoding cost.

## Lab 17. Package a small HLS test

**Goal:** See the difference between compressed video, an HLS segment, and a playlist.

This example creates **12 seconds** of 30 fps video with a 60-frame GOP target (two seconds) and an HLS segment target near two seconds:

```bash
mkdir -p l17_hls
ffmpeg -y -f lavfi \
  -i 'testsrc2=size=640x360:rate=30:duration=12' \
  -map 0:v:0 -an -c:v libx264 -preset medium -crf 23 \
  -g 60 -keyint_min 60 -sc_threshold 0 \
  -x264-params open-gop=0 \
  -f hls -hls_time 2 -hls_playlist_type vod \
  -hls_segment_filename 'l17_hls/seg_%03d.ts' \
  'l17_hls/index.m3u8'

cat l17_hls/index.m3u8
ls -lh l17_hls
```

Play the local playlist in FFplay:

```bash
ffplay -autoexit l17_hls/index.m3u8
```

**What to learn:** Read the `.m3u8` file and list the `.ts` segments. `-hls_time 2` requests a target; actual segment cuts depend on access points and muxer decisions. If local file playlist access is restricted, try serving the folder over HTTP with `python3 -m http.server 8000` and a compatible HLS player. **Think:** Why is a two-second GOP not the same thing as a two-second segment? What must be aligned across 360p and 720p renditions?

## Lab 18. Build a small bitrate–quality ladder experiment

**Goal:** Bring the ideas together: one source, several rates, measurements, and a written decision.

Encode several bitrate targets and collect actual output sizes and rates:

```bash
mkdir -p l18_results
for kbps in 300 500 800 1200 1800; do
  ffmpeg -y -hide_banner -loglevel error \
    -i l02_ref.mkv -an -c:v libx264 -preset medium \
    -b:v "${kbps}k" \
    "l18_results/test_${kbps}k.mp4"
done

for f in l18_results/*.mp4; do
  printf '%s  ' "$f"
  ffprobe -v error -show_entries format=duration,size,bit_rate \
    -of csv=p=0 "$f"
done
```

**What to learn:** For each file, record real bitrate, size, encode time, and a correctly aligned PSNR, SSIM, or VMAF measurement. Put the results in a CSV and draw a quality-versus-bitrate curve. Identify the lowest rate that meets a **rule defined before the experiment**. Look for diminishing returns and dominated candidates. **Write a one-page report:** hypothesis, method, results, two comparable screenshots, problems found, and conclusion.

---

# Appendix A. Read a real x264 log, line by line

This case comes from the actual synthetic-video encoding log used in this onboarding discussion. The source is `testsrc2`, **1920×1080, 30 fps, 300 frames**. FFmpeg encoded it to MP4 with `libx264`, preset `medium`, CRF 18, and 8-bit YUV420p. Read the log first, then use the explanations below.

## A.1 Input and output stream mapping

```text
Stream mapping:
  Stream #0:0 -> #0:0 (wrapped_avframe (native) -> h264 (libx264))
```

FFmpeg maps the generated input picture stream into the H.264 encoder. `wrapped_avframe` is an internal frame representation, not the codec that appears in the final MP4. `#0:0` is an input stream index. It does not mean “frame zero.”

**Engineering meaning:** One video stream enters the encoder. The encoder outputs H.264 coded data. In files with several tracks, explicit stream mapping is important to avoid selecting the wrong track.

## A.2 CPU capabilities, profile, level, and color format

```text
[libx264] using SAR=1/1
[libx264] using cpu capabilities: MMX2 SSE2Fast SSSE3 SSE4.2 AVX FMA3 BMI2 AVX2
[libx264] profile High, level 4.0, 4:2:0, 8-bit
```

- **`SAR=1/1`:** square pixel samples.
- **AVX2, FMA3, and similar names:** CPU instructions that x264 can use to speed up calculations. These are features of the encoding computer, **not requirements of the encoded file**.
- **`profile High`:** an H.264 profile that supports a group of coding tools.
- **`level 4.0`:** decoding-resource constraints for the H.264 stream. It is not a quality measure.
- **`4:2:0`:** reduced chroma sampling relative to luma.
- **`8-bit`:** bit depth of the coded picture samples.

A TV does not need an AVX2 CPU to play this file. It needs a suitable H.264 decoder, perhaps implemented in dedicated hardware.

## A.3 The long x264 option line

Your log reports many options. Group them by purpose:

| Function | Real parameters from the log | What they do |
|---|---|---|
| Motion search | `me=hex`, `subme=7`, `me_range=16`, `chroma_me=1` | Find useful motion predictions |
| Reference selection | `ref=3`, `mixed_ref=1`, `weightp=2`, `weightb=1` | Choose and combine reference data |
| B-pictures | `bframes=3`, `b_pyramid=2`, `b_adapt=1` | Place and organize B-pictures |
| GOP and scene cuts | `keyint=250`, `keyint_min=25`, `scenecut=40`, `open_gop=0` | Choose access-point frequency and group boundaries |
| Bit allocation | `rc=crf`, `crf=18.0`, `qcomp=0.60`, `qpmin=0`, `qpmax=69` | Guide quantization and bitrate choices |
| Future analysis | `rc_lookahead=40`, `mbtree=1` | Use future information and reference value |
| Perceptual tools | `aq=1:1.00`, `psy=1`, `psy_rd=1.00:0.00` | Allocate visible detail differently |
| Reconstruction and coefficient decisions | `deblock=1:0:0`, `8x8dct=1`, `trellis=1` | Improve coding choices and reconstructed pictures |
| Parallel work | `threads=18`, `lookahead_threads=3` | Use CPU threads |

**Engineering meaning:** This was not an abstract “H.264 test.” It used one specific implementation, version, and configuration. Repeating only `-crf 18` may not reproduce the same output after changing presets, FFmpeg builds, or x264 versions.

## A.4 Output file and stream description

```text
Output #0, mp4, to 'test.mp4':
Stream #0:0: Video: h264 (avc1 / 0x31637661), yuv420p(tv, progressive),
1920x1080 [SAR 1:1 DAR 16:9], q=2-31, 30 fps, 15360 tbn
```

- **MP4** is the container.
- **H.264** is the codec; `avc1` identifies the sample entry type.
- **`yuv420p(tv, progressive)`** means planar 4:2:0 samples, limited/TV signal range, and progressive pictures.
- **1920×1080** is the picture size.
- **SAR 1:1; DAR 16:9** means square samples displayed in a 16:9 picture.
- **30 fps** is the output frame cadence.
- **`15360 tbn`** describes a time-base value of 1/15360 second per tick in this output context. It does **not** mean 15,360 fps.
- **`q=2-31`** is an encoder-context field in the output description; do not read it as the measured QP of all output blocks.

Wrong interpretation of `tv` versus full range can create incorrect black and white levels in later processing or playback.

## A.5 Job completion and overall performance

```text
[out#0/mp4] video:12172KiB audio:0KiB subtitle:0KiB ... muxing overhead: 0.034989%
frame=300 fps=142 q=-1.0 Lsize=12176KiB time=00:00:09.93
bitrate=10041.7kbits/s speed=4.69x elapsed=0:00:02.11
```

FFmpeg processed **300 frames**. The coded video payload used about 12,172 KiB. The final file used around 12,176 KiB, approximately **11.9 MiB**. This particular file has no audio or subtitles. The reported container overhead is small.

`fps=142` means about 142 processed frames per wall-clock second for this job. `speed=4.69x` means the whole operation ran about 4.69 times faster than real-time playback. `elapsed=0:00:02.11` means the operation lasted about 2.11 real seconds.

`time=00:00:09.93` is a progress timestamp. It does not, by itself, prove that the 10-second clip is missing its last 0.07 seconds. Frame timestamps describe frame positions, and the final picture has a duration. Check total duration with FFprobe.

`q=-1.0` is not evidence of lossless encoding. It is a generic progress field that does not provide a useful final QP value in this case. Use x264's detailed QP statistics instead.

Why does FFmpeg show approximately `10041.7 kbits/s`, while x264 later says `9970.73 kb/s`? The two components count data and use timing slightly differently. Container overhead and measurement conventions can contribute. Small differences are normal. Large differences deserve investigation.

## A.6 I, P, B counts and average QP

```text
frame I:2     Avg QP: 6.52  size: 64016
frame P:135   Avg QP:19.63  size: 46069
frame B:163   Avg QP:24.63  size: 37522
```

Check the count: `2 + 135 + 163 = 300`. x264 used many P- and B-pictures. Average sizes are in **bytes per picture of that type**: about 64 KB for I, 46 KB for P, and 37.5 KB for B.

The B-pictures have the highest average QP, 24.63. That indicates stronger quantization on average for those pictures. But B-pictures can have good temporal prediction. Therefore, comparing the QP numbers alone does **not** give a reliable comparison of visible quality between I, P, and B.

For real quality investigations, inspect changes across frames and scenes rather than using only whole-video averages.

## A.7 Consecutive B-pictures

```text
consecutive B-frames: 12.7% 40.7% 12.0% 34.7%
```

These percentages describe x264's distribution of runs of consecutive B-pictures in its summary. They are not the fraction of all frames that are B-pictures; you already know that from the I/P/B counts.

`bframes=3` allows up to three consecutive B-pictures under the selected x264 structure. `b_adapt=1` lets the encoder choose where to place them. It does not force every run to contain three B-pictures.

## A.8 Macroblock modes and skip percentages

```text
mb I  I16..4: 82.1% 10.5% 7.4%
mb P  I16..4:  1.1%  2.8% 1.1%  P16..4: 4.2% 3.6% 2.4% ... skip:84.9%
mb B  I16..4:  0.2%  0.5% 1.6%  B16..8: 7.0% 2.8% 0.8% ... skip:86.4%
```

I-pictures use different intra macroblock partitions. P- and B-pictures may combine intra, inter, and skip macroblocks.

For P-pictures, **84.9%** of macroblocks were reported as skip. For B-pictures, **86.4%** were skip. These are internal coding-mode statistics, not percentages of lost pixels, black screen area, or unchanged pixels. Skip prediction may use references other than the immediately previous displayed picture.

The pattern `testsrc2` contains areas that can be predicted efficiently. The remaining regions can still use many bits. So **high skip and almost 10 Mbps are not contradictory**.

## A.9 Transforms and coded coefficients

```text
8x8 transform intra:38.8% inter:18.5%
coded y,uvDC,uvAC intra: 33.2% 40.2% 39.9%
coded y,uvDC,uvAC inter:  3.6%  6.0%  5.5%
```

The first line summarizes use of 8×8 transforms in intra and inter classes. The next lines describe how often nonzero coefficient information appears for luma and chroma categories, including chroma DC/AC groups.

Small inter coded-coefficient percentages are consistent with good temporal prediction and many skipped regions. They are **not** direct perceptual quality scores.

## A.10 Intra prediction directions

```text
i16 v,h,dc,p: 91% 7% 1% 0%
i8 v,h,dc,ddl,ddr,vr,hd,vl,hu: 5% 10% 81% ...
i4 v,h,dc,ddl,ddr,vr,hd,vl,hu: 15% 23% 30% ...
i8c dc,h,v,p: 82% 8% 7% 3%
```

Here `v` means **vertical intra prediction**, not an object moving vertically. `h` means horizontal prediction. `dc` is a prediction based on neighboring sample values, often useful for smooth areas. Other abbreviations are other prediction directions.

The percentages describe how often those prediction modes were chosen for particular block classes. They do not describe physical object trajectories.

## A.11 Reference index distributions

```text
ref P L0: 44.4%  8.1% 29.6% 17.9%
ref B L0: 70.7% 24.5%  4.8%
ref B L1: 95.3%  4.7%
```

`L0` and `L1` are reference lists. The percentages show which **reference indexes** were selected in the corresponding categories. They do **not** tell you how fast objects moved. To study motion speed, examine vectors or other motion measurements.

## A.12 What the log proves — and what it does not

**Supported facts:**

1. There were 300 1920×1080 pictures at 30 fps, encoded as H.264 High, Level 4.0, 8-bit YUV420p.
2. x264 used CRF 18 and settings consistent with preset `medium`.
3. The measured job ran about 4.69 times faster than playback on that particular machine.
4. Final output was about 11.9 MiB with a rate near 10 Mbps.
5. x264 used 2 I, 135 P, and 163 B pictures, with high skip percentages in P and B.
6. Lookahead, AQ, MB-tree, temporal prediction, and CABAC were enabled.

**Unsupported conclusions:**

- We do not know VMAF without an aligned original reference and a measurement.
- We do not know whether 10 Mbps is good or bad for a particular product.
- We cannot infer AV1 or HEVC encoding costs from this H.264 run alone.
- We cannot find exact IDR positions from the total I-picture count.
- We cannot recover physical motion from reference index counts.
- We cannot declare the stream HLS-compliant without packaging and access-point validation.

### A.13 Three mentoring questions

**“Why only two I-pictures?”** Temporal prediction saves bits when many pictures share information. GOP/keyframe settings control the access-point trade-off.

**“Why is CRF 18 but average B-picture QP is 24.63?”** CRF sets a rate-control policy. The encoder chooses actual quantization for individual pictures and blocks. They do not need to have the same number.

**“Why is skip near 85% while bitrate is almost 10 Mbps?”** Many regions can be predicted cheaply, but other regions may be hard. Total bitrate depends on **all** encoded information, not only the skip fraction.

---

# Appendix B. Professional troubleshooting: from symptom to cause

## B.1 The file does not play

First check if the file exists and FFprobe can read its container and streams. Then attempt full decoding:

```bash
ffprobe -hide_banner -i input.mp4
ffmpeg -v error -i input.mp4 -f null -
```

If FFprobe fails, consider truncated headers, invalid container data, wrong path, permissions, or unsupported formats. If FFprobe succeeds but decoding fails, investigate decoder availability and bitstream corruption. If FFplay succeeds but a TV fails, investigate codec, profile, level, pixel format, hardware decoder support, audio, DRM, and delivery packaging.

## B.2 Video is too dark or colors look wrong

Inspect `color_range`, `color_space`, `color_transfer`, `color_primaries`, and `pix_fmt`. Examine all color conversions. Full/limited-range mistakes can damage blacks and whites. Wrong matrices or transfer functions can alter brightness and hue. The display environment may also affect what you see.

## B.3 Duration looks right, but motion stutters

Inspect actual frame timestamps, nominal and average FPS, duplicate or dropped frames, decoder load, reorder buffering, and display cadence. A stream can contain correct metadata while a player still fails to present frames on time.

## B.4 Average bitrate meets the goal, but the player buffers

Inspect segment sizes, rate over short windows, large I-pictures, audio, transport overhead, CDN behavior, network round-trip times, and ABR choices. A whole-file average can hide short bursts that the viewer cannot download in time.

## B.5 VMAF suddenly becomes worse

Check, in order:

1. Are reference and output exactly the same time interval?
2. Are frame counts and PTS aligned, without missing/extra frames?
3. Are resolution and pixel formats handled as intended?
4. Did the model name/version/options change?
5. Did scaling, color range, transfer function, or bit depth change?
6. Did the original reference already contain artifacts that processing reduced?
7. Is the drop limited to a few scenes?
8. Did FFmpeg or libvmaf report any warning or error?

Only then blame codec parameter changes.

## B.6 Blocks appear in fast scenes

Possible causes include insufficient bitrate, a very tight VBV buffer model, rate concentrated near an access point, unsuitable preprocessing, or encoder/preset choices. Inspect QP and packet sizes over time. Try controlled changes to the available bitrate or VBV restrictions, and measure again.

## B.7 Banding appears even when VMAF is high

Check with CAMBI and visual review. Inspect input/output bit depth, transfer/range conversions, and quantization. The banding may already exist in the source. VMAF does not represent every visual artifact equally well.

## B.8 Cutting or joining creates glitches

Check decodable access points, consistent codec parameters, continuous PTS/DTS, and correct muxing. Appending arbitrary MP4 file bytes is not valid media concatenation. Stream-copy cuts may need nearby keyframes. Segment boundaries must respect reference dependencies.

## B.9 The encoder is too slow

Separate the costs of demux, decode, filtering, encoding, quality measurement, and I/O. Use simpler controlled tests to find the expensive stage. Faster presets are one option. Avoiding repeated decoding, unneeded scaling, or duplicate disk reads may help as well. In WSL, check CPU load on Windows and where temporary files are stored.

## B.10 Live latency is too high

Build a timeline across capture, encoder, ingest, packager, CDN, player, and display. Check B-picture reordering, lookahead, VBV, segment/part duration, media availability, player buffer, and decoder delay. Do not simply say “FFmpeg adds 15 seconds” before measuring each stage.

## B.11 How to report an incident to an engineering team

Include the expected and actual result, a small reproducible sample, FFmpeg command and version, complete logs, `ffprobe` JSON, affected device/client, frequency, impact, initial hypotheses, and tests already performed. For color bugs, attach the conversion chain. For bitrate issues, include short-window measurements. For quality issues, include the reference, output, and comparison method.

---

# Appendix C. Learning plan and self-assessment

## C.1 What you should know after week one

You should be able to describe the full path from a source to a displayed picture. Explain why **MP4 is a container** and **H.264 is a codec**. Explain 1920×1080, 30 fps, YUV420p, and 8-bit. Identify the ideas behind I, P, and B-pictures and why GOP boundaries matter. Generate synthetic video with FFmpeg, play it with FFplay, and inspect it with FFprobe.

**Practice question:** “I have a ten-second 1080p30 MP4. What do I know, and what must I still check?” A careful engineer does not assume frame count if the stream is VFR and checks codec, audio, color, bitrate, profile, actual timestamps, and decoder compatibility.

## C.2 What you should know after week two

Explain why codecs use blocks, intra/inter prediction, motion vectors, and residuals. Explain what quantization removes. Explain why **CRF is not QP**, and why two CRF 20 files can have different sizes. You should also be able to read `frame I/P/B`, `Avg QP`, `skip`, `rc=crf`, `aq`, and `mbtree` in an x264 log without claiming that the log proves more than it does.

**Practice question:** “Two videos use CRF 20, but one is three times larger. Is something wrong?” Not necessarily. The videos may have different complexity. CRF does not set a fixed output size. Verify the rest of the conditions.

## C.3 What you should know after week three

You should be able to create a fair preset or codec comparison. Choose a reference, keep variables stable, run PSNR/SSIM or VMAF correctly, and explain the limits of each metric. Distinguish VOD from live operation. Describe why GOP and segment alignment matters for an ABR ladder.

**Practice question:** “Codec A gives VMAF 96 and Codec B gives VMAF 95. Should we always choose A?” No. We need information about bitrate, preprocessing, model version, content, viewing conditions, visual artifacts, encoding/decoding cost, latency, and device support.

## C.4 Twenty review questions

Try to answer without looking at the solutions. Some are engineering explanations, not one-word facts.

1. What is the difference between a codec and a container?
2. Why is H.264 not the same thing as x264?
3. How is 8-bit YUV420p different from RGB24?
4. What does VFR mean?
5. Why do both PTS and DTS exist?
6. Can an I-picture be lossy-compressed?
7. Can a P-picture contain intra-coded regions?
8. Why can B-pictures improve compression but add reordering work?
9. Why does an I-picture alone not prove that a segment is independently decodable?
10. What is a skip macroblock? Is it a lost frame?
11. What problem does temporal prediction solve?
12. Which common processing step causes most deliberate loss in classical hybrid codecs?
13. What generally happens when QP increases?
14. Why is CRF 18 not the same as QP 18?
15. What is the difference between an ABR target and a VBV constraint?
16. What resources does lookahead use?
17. What does MB-tree try to improve?
18. Why do we create a lossless reference?
19. Why should VMAF not be the only quality rule?
20. Why can a good VOD encoder configuration be wrong for live video?

### Answers with short reasons

1. A codec defines coded media and reconstruction. A container groups streams and playback metadata.
2. H.264 is the compression standard; x264 is an encoder implementation.
3. YUV420p uses lower chroma sampling than full-resolution RGB. It does not keep the same samples as RGB24.
4. Frame presentation intervals may vary. Average fps does not define every frame's timestamp.
5. Some pictures must be decoded before pictures displayed earlier, especially with reordering/reference structures.
6. Yes. Intra means no temporal prediction, not lossless coding.
7. Yes. The picture type allows inter prediction but individual regions can use intra.
8. References on both sides can help prediction. Those references may need to be decoded and buffered before a B-picture is shown.
9. An I-picture may be non-IDR. Other pictures may still depend on older references; access-point rules and packaging matter.
10. Skip is a coding mode that uses inferred prediction information without an explicit coded residual. It is not a lost image.
11. It avoids sending full new pixel descriptions for regions that can be predicted from other pictures.
12. Usually quantization, with possible earlier loss from chroma conversion, resizing, or other processing.
13. Quantization becomes stronger; this often saves bits while losing more detail.
14. CRF is a global rate-control policy. The encoder chooses actual QP across pictures and blocks.
15. ABR aims for an average. VBV limits bitrate/buffering behavior over time in a specific model.
16. CPU work, memory, and possible delay because upcoming pictures are examined.
17. It uses the future reference importance of blocks to guide bit allocation.
18. It avoids adding loss from the reference file's video codec. It cannot reverse preprocessing done earlier.
19. A metric may miss certain artifacts or viewing conditions, and it cannot measure player stalls or delivery latency.
20. VOD can analyze all content. Live has strict limits on lookahead, CPU time, buffering, and end-to-end delay.

## C.5 Open engineering scenarios

### Scenario A — A film for an OTT catalog

You have a high-quality master and many hours for processing. You want good visible quality at a low CDN cost and must support many TV generations. A reasonable plan is to build a widely compatible H.264 ladder and evaluate HEVC or AV1 where target devices support them. Use several bitrate points and examine each rendition's quality, segment peaks, and playback compatibility. Two-pass or content-aware encoding may be useful when the improvement justifies processing cost.

**Explain your decision:** What would you measure before approving a more expensive codec? How would you know if a lower bitrate actually helps the people using the service?

### Scenario B — A live football channel

The service needs low latency, continuous operation, and reliable playback. First identify the actual latency budget and distribution protocol. Then choose acceptable encoder delay, GOP/access-point structure, bitrate and VBV limits, and player buffering strategy. Monitor segments and end-to-end latency. Use fast signals for control and deeper visual quality tests afterward.

**Explain your decision:** Why does a slower preset or longer lookahead become difficult in live operation, even if it gives better rate–distortion results offline?

### Scenario C — A strange VMAF drop

Your new pipeline has a much lower score than the previous one. Do not change encoder parameters immediately. Confirm the reference and distorted outputs have the same content and correct frame alignment, resolution, color conversion, and metric model. Look at per-frame scores and the actual picture.

**Explain your decision:** Which two checks would you run first? What observations would support a real codec quality problem?

### Scenario D — A synthetic dataset for better bitrate predictions

Generate videos with controlled motion, spatial detail, noise, and scene changes. Encode each one at several rates and build a rate–distortion curve. Learn whether simple source features predict the required bitrate. Then test the predictor on real videos that were not used for training.

**Explain your decision:** Why does a moving geometric square not represent the full complexity of grass, hair, crowds, moving water, or film grain?

---

# Appendix D. Technical glossary

This glossary keeps the names used by FFmpeg, encoder logs, and technical documentation. Read it as a reference, not as a list to memorize.

| Term | Simple meaning |
|---|---|
| **ABR (Average Bitrate)** | Rate-control method that aims for an average number of bits per second. |
| **ABR ladder** | Set of alternative video renditions offered to an adaptive player. |
| **AQ (Adaptive Quantization)** | Encoder tool that allocates quantization differently across parts of a picture. |
| **Artifact** | An unwanted visible change or error in an image. |
| **AV1** | Modern video codec from AOMedia. |
| **AVC (H.264)** | Very common video compression standard. |
| **Banding** | Visible steps where color or brightness should change smoothly. |
| **B-frame / B-picture** | Picture that can use two reference lists for temporal prediction. |
| **Bit depth** | Number of bits used to represent a sample's value. |
| **Bitrate** | Amount of coded data per second. |
| **Bitstream** | Compressed data in the format defined by a codec. |
| **Blocking** | Visible square or rectangular image patterns caused by coding or transmission problems. |
| **CABAC** | Context-adaptive entropy coding method used in H.264 and related designs. |
| **CAMBI** | Metric designed to measure image banding. |
| **CBR (Constant Bitrate)** | Rate behavior intended to remain constant under a particular definition or transport model. |
| **CDN** | Network of servers used to deliver content closer to users. |
| **Chroma** | Color-difference information. |
| **Chroma subsampling** | Using fewer spatial color samples than luma samples, as in 4:2:0. |
| **Codec** | Rules for coding and decoding audio or video. |
| **Container** | File or stream format that holds media streams and related metadata. |
| **CPB** | Coded Picture Buffer used in the hypothetical decoder model. |
| **CRF** | Encoder rate-control mode for a chosen quality–size trade-off. |
| **CFR** | Constant Frame Rate: regular intervals between displayed frames. |
| **CTU** | Coding Tree Unit used in HEVC. |
| **DASH** | MPEG-DASH, a format for HTTP adaptive streaming. |
| **DAR** | Display Aspect Ratio: displayed picture shape. |
| **Deblocking** | Filter that reduces some visible boundaries between coded blocks. |
| **Decoder** | Software or hardware that reconstructs media from a coded stream. |
| **Demuxer** | Component that separates streams from a container. |
| **DTS** | Decoding Timestamp: decoding schedule information. |
| **Entropy coding** | Efficient representation of coded symbols using probabilities or contexts. |
| **FFmpeg** | Software tools and libraries for multimedia processing. |
| **FFplay** | Simple media player included in many FFmpeg installations. |
| **FFprobe** | Tool for inspecting media streams and metadata. |
| **FFV1** | Lossless video codec useful for reference files. |
| **Frame** | One video picture at a given presentation time. |
| **Frame rate / fps** | Number of pictures presented per second, or a rate describing that cadence. |
| **GOP** | Group of Pictures: picture/reference structure around access points. |
| **H.264** | Same video standard family as AVC. |
| **HEVC (H.265)** | Video codec newer than H.264, with different coding tools. |
| **HLS** | HTTP Live Streaming, an HTTP playlist/segment system. |
| **HRD** | Hypothetical Reference Decoder model for timing and buffers. |
| **I-frame / I-picture** | Picture coded without temporal prediction from other pictures. |
| **IDR** | H.264 access point with defined reference-refresh behavior. |
| **Inter prediction** | Prediction using decoded reference pictures. |
| **Intra prediction** | Prediction using reconstructed neighbors within the same picture. |
| **Keyframe** | Access-point-related picture or label; exact meaning depends on context. |
| **Latency** | Delay between an event or input and the resulting output or playback. |
| **libx264** | Software H.264 encoder library used by FFmpeg. |
| **libx265** | Software HEVC encoder library used by FFmpeg. |
| **libsvtav1** | Software AV1 encoder library used by FFmpeg. |
| **Limited range** | Video signal convention using a defined nominal range smaller than full digital code range. |
| **Lookahead** | Analysis of future pictures before final coding decisions. |
| **Luma (Y′)** | Encoded signal component related to brightness. |
| **Macroblock** | Important 16×16 luma coding unit in H.264. |
| **Manifest** | File describing streams, representations, and segments for playback. |
| **MB-tree** | x264 bit-allocation method using future reference importance. |
| **MSE** | Mean Squared Error between compared samples. |
| **Motion compensation** | Building predicted samples from reference frames and motion information. |
| **Motion estimation** | Encoder search for useful reference regions and motion vectors. |
| **Motion vector** | Coded displacement used to build a temporal prediction. |
| **MP4** | Common multimedia container based on the ISO media file family. |
| **Muxer** | Component that packs encoded streams into a container. |
| **NAL unit** | H.264 bitstream organization unit. |
| **Origin server** | Server or system holding primary media for delivery. |
| **P-frame / P-picture** | Picture with prediction choices that include inter references. |
| **Packet** | Unit of coded media data passed through a demux/mux/decode pipeline. |
| **Pixel format** | Layout and representation of samples, such as `yuv420p` or `rgb24`. |
| **Playback buffer** | Media/data kept by a player to handle download and decode variation. |
| **PPS / SPS** | H.264 parameter sets needed to interpret coded data. |
| **Preset** | Named group of encoder speed/efficiency settings. |
| **PSNR** | Peak Signal-to-Noise Ratio, a pixel-error measure in dB. |
| **PTS** | Presentation Timestamp: when a decoded picture should be shown. |
| **QP** | Quantization Parameter: a control on quantization strength. |
| **Rate control (RC)** | Encoder logic for allocating quantization and bits. |
| **Rate–distortion (R–D)** | Trade-off between bitrate and error/quality loss. |
| **Raw video** | Uncompressed pixel samples, often without sufficient format metadata. |
| **Reference frame** | Reconstructed picture used to predict another picture. |
| **Remux** | Change media packaging while keeping compatible compressed streams. |
| **Residual** | Difference between source data and a chosen prediction. |
| **Residual energy** | A measure of the amount of prediction error; often based on squared differences. |
| **Ringing** | Halos or ripples around edges from certain transforms, coding, or resizing. |
| **SAR** | Sample Aspect Ratio: shape of stored pixel samples. |
| **Scenecut** | Scene-change detection that may influence intra/keyframe decisions. |
| **SEI** | Supplemental Enhancement Information in a codec bitstream. |
| **Segment** | Deliverable piece of media, not necessarily the same as a GOP or scene. |
| **Skip** | Predictive mode where explicit residual data is avoided under codec rules. |
| **Slice** | H.264 coded picture organization unit. |
| **SSIM** | Structural Similarity image comparison metric. |
| **Stream copy** | Copy compatible compressed stream data without decode/re-encode. |
| **Subme** | x264 setting for subpixel refinement and other related decisions. |
| **Time base** | Rational unit used for timestamps. |
| **Transcode** | Convert a coded stream to another representation, usually decode then encode. |
| **Transform** | Converts sample patterns such as residuals into coefficient data. |
| **VBR** | Variable Bitrate: rate changes over time. |
| **VFR** | Variable Frame Rate: uneven intervals between presented frames. |
| **VBV** | Video Buffering Verifier model for rate and buffer limits. |
| **VMAF** | Full-reference perceptual video quality model developed at Netflix. |
| **Y′CbCr** | Digital video representation with luma and color-difference components. |

---

# Appendix E. Quick reference for the first day on the job

## E.1 Before changing a production file

```bash
# Show the container, stream summary, and duration
ffprobe -hide_banner input.mp4

# Show detailed video and audio information as JSON
ffprobe -v error -show_streams -show_format -of json input.mp4

# Decode everything without writing an output file; report decode errors
ffmpeg -v error -i input.mp4 -f null -

# Check which encoders and filters actually exist in this installation
ffmpeg -hide_banner -encoders
ffmpeg -hide_banner -filters

# Read options available for x264 through this FFmpeg build
ffmpeg -hide_banner -h encoder=libx264
```

A good investigation sequence is: **reproduce the failure → keep a small sample → inspect the input → identify the failing stage → make one controlled change → measure again → document the result**.

## E.2 Common encoding command templates

### H.264 quality–size test

```bash
ffmpeg -i input.mp4 -map 0:v:0 -an \
  -c:v libx264 -preset medium -crf 20 -pix_fmt yuv420p \
  output_h264.mp4
```

### H.264 average bitrate with VBV constraints

```bash
ffmpeg -i input.mp4 -map 0:v:0 -an \
  -c:v libx264 -preset medium -b:v 2500k \
  -maxrate 3000k -bufsize 6000k output_vbv.mp4
```

### H.264 regular closed-GOP style for two-second access points at 30 fps

```bash
ffmpeg -i input.mp4 -map 0:v:0 -an \
  -c:v libx264 -preset medium -b:v 2500k \
  -g 60 -keyint_min 60 -sc_threshold 0 \
  -x264-params open-gop=0 \
  output_gop.mp4
```

This is an example for a known 30 fps source. `-g 60` counts **frames**, not seconds. For VFR or a different frame rate, first decide the time behavior you need. Always verify actual access points.

### Preserve the decoded source samples with a lossless codec

```bash
ffmpeg -i input.mp4 -map 0:v:0 -an -c:v ffv1 reference.mkv
```

The output prevents additional lossy coding after the input is decoded. It **cannot** restore details lost in the original compressed `input.mp4`.

### Optional audio-and-video synthetic check

```bash
ffmpeg -y \
  -f lavfi -i 'testsrc2=size=640x360:rate=30:duration=6' \
  -f lavfi -i 'sine=frequency=440:sample_rate=48000:duration=6' \
  -map 0:v:0 -map 1:a:0 \
  -c:v libx264 -preset medium -crf 23 -pix_fmt yuv420p \
  -c:a aac -b:a 128k -shortest synthetic_with_audio.mp4

ffprobe -v error -show_entries stream=codec_type,codec_name,duration \
  -of default=noprint_wrappers=1 synthetic_with_audio.mp4
```

This extra exercise lets you check video/audio stream mapping and durations. It also proves that a synthetic-video pipeline can include synthetic audio.

## E.3 Ten habits that prevent expensive mistakes

1. **Never confuse container and codec.** Inspect the streams.
2. **Never confuse CRF, QP, bitrate, and VMAF.** They describe different things.
3. **Do not call a smaller file a better result without measuring quality.**
4. **Do not interpret skip as lost pixels.** It is a prediction mode.
5. **Do not assume every I-picture is an IDR or an independent HLS access point.**
6. **Do not ignore pixel range and color metadata.** Many “codec bugs” are color-chain bugs.
7. **Never compare quality without checking temporal and spatial alignment.**
8. **Do not use only the average score for variable content.** Check weak scenes and percentiles.
9. **Do not treat synthetic test videos as a complete substitute for real content.**
10. **Do not design encoding separately from players, networks, and devices.**

## E.4 A reusable experiment record

Copy this template and fill in real values. Write acceptance rules before running the experiment.

```yaml
experiment_id: EXP-001
hypothesis: 'The slow preset improves quality at equal average bitrate'
reference:
  path: l02_ref.mkv
  codec: ffv1
  width: 640
  height: 360
  fps: '30/1'
  pixel_format: yuv420p
  color_space: unknown_or_recorded
encoder:
  implementation: libx264
  ffmpeg_version: 'copy from ffmpeg -version'
  preset: slow
  rate_control: ABR
  target_kbps: 900
  gop_frames: documented
output:
  path: result.mp4
  actual_kbps: to_measure
  size_bytes: to_measure
  encode_seconds: to_measure
quality:
  metric: VMAF_or_SSIM
  model_version: record_if_using_VMAF
  reference_alignment: checked
  value: to_measure
visual_notes:
  - 'Record visible artifacts and their timestamps'
acceptance_criteria:
  - 'Record the decision rule before encoding'
conclusion: to_write
```

## E.5 What to do when a statement sounds too simple

Be careful with claims such as “HEVC always gives 50% better compression” or “VMAF 95 is invisible.” Ask for the missing conditions: bitrate, quality metric, model version, resolution, content, preset, encoder version, CPU budget, decoder support, viewer, display, and tests.

A small result with clear evidence is more useful than a large claim that fails when someone tests a different scene.

---

# Technical sources and further reading

This guide is an original teaching explanation. The sources below provide primary documentation and technical detail. References follow APA 7 conventions where applicable. For living technical documentation without a stable publication date, **n.d.** means *no date*. Visit the official websites when you need the current behavior of a command, codec, or streaming protocol.

1. FFmpeg Project. (n.d.). *FFmpeg documentation*. https://ffmpeg.org/ffmpeg.html
2. FFmpeg Project. (n.d.). *FFmpeg codecs documentation*. https://ffmpeg.org/ffmpeg-codecs.html
3. FFmpeg Project. (n.d.). *FFmpeg filters documentation*. https://ffmpeg.org/ffmpeg-filters.html
4. FFmpeg Project. (n.d.). *FFprobe documentation*. https://ffmpeg.org/ffprobe.html
5. VideoLAN. (n.d.). *x264: H.264/MPEG-4 AVC encoder*. https://www.videolan.org/developers/x264.html
6. Netflix. (n.d.). *VMAF* [Computer software]. GitHub. https://github.com/Netflix/vmaf
7. Netflix. (2026). *Models (v1)* [Technical documentation]. GitHub. https://github.com/Netflix/vmaf/blob/master/resource/doc/models_v1.md
8. Netflix. (n.d.). *Using VMAF with FFmpeg* [Technical documentation]. GitHub. https://github.com/Netflix/vmaf/blob/master/resource/doc/ffmpeg.md
9. Pantos, R., & May, W. (2017). *HTTP Live Streaming* (RFC 8216). Internet Engineering Task Force. https://www.rfc-editor.org/rfc/rfc8216.html
10. Apple. (n.d.). *HLS authoring specification for Apple devices*. Apple Developer Documentation. https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices/
11. International Telecommunication Union. (n.d.). *Recommendation ITU-R BT.709: Parameter values for the HDTV standards for production and international programme exchange*. https://www.itu.int/rec/R-REC-BT.709
12. International Telecommunication Union. (2025). *Recommendation ITU-R BT.2100: Image parameter values for high dynamic range television for use in production and international programme exchange*. https://www.itu.int/rec/R-REC-BT.2100-3-202502-I
13. Alliance for Open Media. (n.d.). *AV1 video codec*. https://aomedia.org/specifications/av1/
14. Alliance for Open Media. (n.d.). *AV1 bitstream and decoding process specification*. https://aomediacodec.github.io/av1-spec/
15. Alliance for Open Media. (n.d.). *SVT-AV1 encoder documentation* [Computer software documentation]. https://github.com/AOMediaCodec/SVT-AV1

## Important update: VMAF v1 and model choice

As of June 2026, the VMAF project documents a new family of **v1 models** in addition to earlier v0 models. For example, it documents separate models for standard 1080p viewing, phone viewing, and 4K viewing. Some model configurations may even use different score ranges, including one 4K v1 model with a documented **0–110** range. The project also recommends evaluating SDR with appropriate 10-bit precision for better banding behavior in v1. **Do not assume every VMAF model has exactly the same scale or meaning.** Always save the model name and preprocessing choices with your result. See the Netflix *Models (v1)* document above.

## Maintenance note

The general ideas in this guide are stable, but FFmpeg builds, encoder defaults, supported filters, optional libraries, VMAF models, device support, and delivery profiles change. Before using a command in production, check the versions and test the real target system. A short command working in one Ubuntu installation is not a guarantee for every build.

---

## AI assistance and document attribution

This document was prepared with the assistance of **ChatGPT (OpenAI, GPT-6)**, based on the learning goals, instructions, and context provided by the author. AI assistance was also used during the technical review.

- **Tool:** ChatGPT
- **Model:** GPT-6
- **Developer:** OpenAI
- **Date of generation and revision:** October 18, 2026
- **Website:** https://chatgpt.com/

AI-generated or AI-revised content may contain errors or incomplete explanations. Verify important technical claims and production commands against the original specifications, official documentation, and your own tests.

**End of guide.**
