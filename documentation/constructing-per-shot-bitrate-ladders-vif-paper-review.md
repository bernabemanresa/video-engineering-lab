# Paper Review: Constructing Per-Shot Bitrate Ladders Using Visual Information Fidelity

**Document type:** Scientific paper summary and critical analysis  
**Date:** October 18, 2026  
**Language:** Simple English for international readers  
**Source:** Durbha and Bovik (2025)  
**Status:** Personal learning notes. This is **not** an independent reproduction or validation of the experiment.

> **Scope:** This document explains one published scientific paper. It focuses on the authors' research question, method, experiment, results, and limits. It does not describe a new encoding system or a proposed project.

## 1. Paper information

**Title:** *Constructing Per-Shot Bitrate Ladders Using Visual Information Fidelity*  
**Authors:** Krishna Srikar Durbha and Alan C. Bovik  
**Journal:** *IEEE Transactions on Image Processing*  
**Year:** 2025  
**Volume and pages:** 34, 7093–7108  
**DOI:** [10.1109/TIP.2025.3625750](https://doi.org/10.1109/TIP.2025.3625750)

**Links:**

- [Published paper at IEEE](https://ieeexplore.ieee.org/document/11223695/)
- [Free text on arXiv](https://arxiv.org/html/2408.01932)
- [arXiv paper page](https://arxiv.org/abs/2408.01932)
- [Code published by the authors](https://github.com/krishnasrikard/Constructing-Per-Shot-Bitrate-Ladders-using-Visual-Information-Fidelity)

The APA 7 reference is at the end of this document. The paper is the main source for every reported experimental result below (Durbha & Bovik, 2025).

## 2. Short summary

Video streaming services offer several versions of the same video. Some versions use less bitrate. Others use more bitrate and may offer better image quality.

These versions are often organized in a **bitrate ladder**: a list of bitrate and resolution choices.

A fixed ladder uses similar choices for many different videos. But a video with little detail may not need the same settings as a video with a lot of detail and motion.

The authors study how to build a different ladder for each **shot**. A shot is a short video part with no camera cut inside it.

One way to find the best ladder is to encode the same shot many times, measure the quality of every result, and choose the best points. This is expensive.

The authors ask a different question:

**Can a machine-learning model predict useful bitrate and resolution choices from the original video, without encoding many test versions of every new shot?**

Their answer is promising within their experiment. Models using video characteristics from the source produced ladders close to a reference ladder built with many real encodes. The prediction step did not need test encodes or VMAF calculations on compressed versions.

**Important difference:** the models still needed real encodes and VMAF measurements **during training**. The paper does not remove encoding from the whole process. It removes exploratory encoding from the prediction step for a new shot.

## 3. Basic ideas needed to understand the study

### 3.1. Bitrate ladder

A bitrate ladder is a list of video versions for adaptive streaming.

For example, a service could offer a low-resolution version at a small bitrate and a high-resolution version at a larger bitrate. The player can move between versions when the network changes.

The paper looks for a **content-dependent** ladder. This means that the best resolution for a given bitrate can change from one shot to another.

### 3.2. Rate–quality (RQ) curve

An **RQ curve** shows how video quality changes when bitrate changes.

- **Rate** means the number of bits used per second.
- **Quality** means the quality estimate. The study uses VMAF.

More bitrate often improves quality, but the improvement is not always the same. Different resolutions can produce different RQ curves.

### 3.3. Pareto front and convex hull

Imagine that we encode one shot at many resolutions and CRF values. We obtain many points with different bitrates and VMAF scores.

Some points are poor choices. Another encode may give better quality at the same bitrate, or similar quality using fewer bits.

The useful boundary of these choices is called the **Pareto front**. In this paper, the authors also use the term **convex hull** when describing the best operating points.

Finding this boundary with real encodes is the expensive reference method.

### 3.4. VMAF

**VMAF** is a model used to estimate perceived video quality. It compares a processed video with its reference video.

In this experiment, the authors encoded the source videos, scaled the encoded versions back to the original 4K size, and then measured VMAF. This gave the quality values used for training and evaluation.

At prediction time, the new shot has no encoded test version. The model **predicts** VMAF rather than measuring it on a new encode.

### 3.5. VIF

**VIF** means *Visual Information Fidelity*. It is a method based on how much visual information may be available to a viewer.

The authors extract **VIF-related features** at different image scales and frequency bands. They also test features based on differences between nearby frames.

A key detail: the VIF-related features in this method come from the **source video only**. This is not the same as running a full-reference VIF quality comparison on a source and an encoded video.

### 3.6. BD-rate and BD-VMAF

The authors use **Bjøntegaard Delta** measures to compare two RQ curves over a quality range.

- **BD-rate:** average difference in bitrate between two curves. A **negative** value means the tested method needs less bitrate than the comparison method, for comparable quality.
- **BD-VMAF:** average quality difference between the curves. A **positive** value means a quality gain.

These are comparisons between curves. A BD-rate gain of 20% does **not** mean that every file becomes exactly 20% smaller.

## 4. The research method

The paper studies two prediction tasks.

### 4.1. Task A: Predict quality from bitrate

The model receives:

1. Features extracted from the original video.
2. A candidate bitrate.
3. A candidate output resolution.

It predicts the VMAF score expected for that choice.

At each bitrate step, the method compares the predicted quality at different resolutions. It selects the resolution with the highest predicted VMAF.

This creates a **bitrate ladder**.

### 4.2. Task B: Predict bitrate from quality

The model receives:

1. Features extracted from the original video.
2. A target VMAF score.
3. A candidate output resolution.

It predicts the bitrate needed to reach that quality.

At each quality step, the method selects the resolution with the lowest predicted bitrate.

This creates a **quality ladder**.

### 4.3. Training and prediction are different stages

**Training stage:**

1. Read the source videos.
2. Create many encoded versions at different resolutions and CRF values.
3. Measure their real bitrate and VMAF.
4. Extract features from the source video.
5. Train models to connect these features and encoding settings with bitrate or quality.

**Prediction stage for a new shot:**

1. Extract source-video features.
2. Give candidate bitrate or quality values and resolutions to the trained model.
3. Predict the missing values.
4. Build a ladder from the predictions.

No exploratory encodes or compressed-video VMAF measurements are needed in the second stage. The cost of extracting the source features remains.

The authors tested **Extra-Trees**, **XGBoost**, and **Random Forest** regressors. Extra-Trees performed best on almost every task in their comparisons.

## 5. What features did they test?

The study compares different ways to describe the original video.

| Feature | Meaning in simple English |
|---|---|
| **SI — Spatial Information** | How much spatial detail and strong edges the video contains. |
| **TI — Temporal Information** | How much the image changes between nearby frames. |
| **GLCM — Gray-Level Co-occurrence Matrix** | Patterns in how gray values appear next to each other; useful for texture. |
| **TC — Temporal Coherence** | How stable visual patterns are over time. |
| **DCT texture features** | How image detail and variation are distributed in different frequency components. |
| **Color and luminance features** | Statistical information about brightness and color. |
| **VIF-related features** | Measures of visual information taken at different scales and bands of the source frames. |
| **Frame-difference features** | Measures based on changes between consecutive frames. |

The authors define three main sets of *low-level features*:

| Set | Number of features | Main use |
|---|---:|---|
| **LLF1** | 93 | General low-level features; also used for predicting points where resolutions cross. |
| **LLF2** | 96 | Low-level features, including features that depend on the candidate bitrate; used for quality prediction. |
| **LLF3** | 96 | Low-level features, including features that depend on the target VMAF; used for bitrate prediction. |

They also compare nine different **VIF feature sets**, called `VIFF1` to `VIFF9`. Some sets use only spatial information. Others add temporal differences.

Finally, they test **ensembles**: models that use low-level and VIF-related features together.

**Important:** The paper's DCT *texture energy* features are not the same thing as motion-compensated residual energy in a video encoder. Both involve signal analysis, but they measure different things. The paper does not test a motion-compensated residual-energy proxy.

## 6. The experiment: data and encoder settings

### 6.1. Source data

The authors use the **BVT-100 4K** dataset.

| Property | Value reported in the paper |
|---|---|
| Number of sequences | **100** |
| Source resolution after preparation | **3840 × 2160 (UHD)** |
| Frames per sequence | **64** |
| Content | One shot per sequence, with no scene cut |
| Chroma format | **4:2:0** |
| Typical frame rate | Most sequences use **60 fps** |
| Typical bit depth | Most sequences use **10 bits** |
| Training sequences | **70** |
| Validation sequences | **10** |
| Test sequences | **20** |

The dataset includes video material from several public sources. The authors kept video titles separate across training, validation, and test groups to reduce content overlap.

For their reported evaluation, they combined the 10 validation sequences and the 20 test sequences: **30 sequences not used for training**.

A 64-frame sequence at 60 fps lasts only about **1.07 seconds**. This is an important fact when interpreting the results.

### 6.2. Encoding setup

The authors first explored a wider space of eight resolutions and CRF values from **15 to 45**.

For the main experiment, they kept these settings:

| Parameter | Main experiment |
|---|---|
| Encoder | **HEVC / libx265** |
| Preset | **medium** |
| Resolutions | **2160p, 1440p, 1080p, 720p, 540p, 432p** |
| CRF values | **16–35**, plus **37, 39, 41** |
| VMAF range used | **15 to 95**, inclusive |
| Scaling method | **Lanczos** |
| Quality measurement | VMAF after scaling encoded video to **3840 × 2160** |

This means there were **6 resolutions × 23 CRF settings = 138 possible encode settings per source sequence** in the main setting grid, before filtering results by the VMAF rule. This total is a calculation from the settings described in the paper.

The authors use **CRF (Constant Rate Factor)** for constant-quality encoding. The resulting bitrate is measured; it is not necessarily a fixed bitrate chosen before encoding.

### 6.3. Why only VMAF values from 15 to 95?

The authors observed many very low quality results at small resolutions or large CRF values. At the other end, quality increases could become small.

They therefore focused on a VMAF range from **15 to 95**.

This is a study design choice. It affects what the model learns and the range in which the reported results should be interpreted.

## 7. How did the authors evaluate the method?

They compared the predicted ladders with two important baselines:

1. **Fixed ladder:** a set of bitrate and resolution choices that does not adapt to each shot.
2. **Reference ladder:** a ladder sampled from the best RQ points found by exhaustive encoding and VMAF evaluation.

They also compared different feature groups and methods for predicting **cross-over points**. A cross-over point is a place where the preferred resolution changes along the rate–quality curves.

They measured **BD-rate**, **BD-VMAF**, and how close the predicted ladders were to the reference method.

The bitrate ladder uses 13 bitrate steps, from **500 to 15,000 kbps**. The quality ladder uses 13 VMAF steps, from **25 to 92.5**.

The study also applied correction rules to avoid unreasonable resolution changes across ladder steps and checked the monotonic behavior of the predicted curves.

## 8. Main results

### 8.1. Bitrate ladders

The following values come from the paper's results for the validation and test sequences (Table IX and related text). **Negative BD-rate is better** when comparing with the fixed ladder.

| Method | Mean BD-rate vs. fixed ladder | Mean BD-rate vs. reference ladder |
|---|---:|---:|
| **Reference ladder**, built from exhaustive encodes | **−20.63%** | — |
| **Metadata only**: bitrate and resolution | −9.256% | +14.053% |
| **LLF2**: low-level features | **−20.429%** | **+0.439%** |
| **VIFF9**: VIF-related features | −17.764% | +3.529% |
| **LLF2 + VIFF6**: combined features | **−20.688%** | **+0.347%** |
| **LLF1 cross-over prediction** | −20.430% | +1.342% |

**How to read this table:**

- The fixed ladder is the baseline.
- The reference ladder improves mean BD-rate by **20.63%** against that baseline.
- The LLF2 model improves mean BD-rate by **20.429%** against the same baseline.
- LLF2 has an average BD-rate **loss of 0.439% relative to the reference ladder**. This is a curve comparison, not a 0.439-point difference in VMAF.
- Combining LLF2 with VIFF6 gives an even smaller mean difference from the reference, but computing VIF features has extra cost.

The reported **standard deviation for LLF2's BD-rate vs. the fixed ladder is 16.747 percentage points**. Results differ considerably between sequences; the mean alone does not show every individual result.

### 8.2. Quality ladders

The authors also built ladders around target quality levels instead of target bitrate levels (Table X).

Examples:

| Method | Mean BD-rate vs. fixed ladder | Mean BD-rate vs. reference ladder |
|---|---:|---:|
| **LLF3**: low-level features | −20.546% | +0.615% |
| **LLF3 + VIFF4** | −20.727% | +0.294% |
| **LLF1 cross-over VMAF prediction** | −20.863% | −0.129% |

The slightly negative result against the reference in the last row does **not** prove that prediction is better than the complete Pareto front. The authors explain that sampling the reference curve can introduce small differences and noise.

### 8.3. What these numbers show

The most important comparison is not only between two VMAF predictions. It is between the **final ladder choices**.

A model can predict individual quality values with some error and still select good resolutions at most ladder steps.

The results suggest that features from the original source contain enough useful information to build strong bitrate and quality ladders in this test setting.

They also show that **more features do not always give a large practical improvement**. A relatively affordable low-level feature set performs close to more complex combinations.

## 9. Computational cost

The authors measured execution times on an **AMD Ryzen 9 5950X**. The times below are approximate and describe their own implementations and hardware (Table XI).

| Operation | Mean time reported |
|---|---:|
| One compression operation | **0.42 min** |
| Upscaling and VMAF evaluation | **0.67 min** |
| Extract LLF1 low-level features | **6 min** |
| Extract VIFF9 features in Python | **40 min** |
| Extra-Trees prediction | **0.1 min** |
| Build the exhaustive reference ladder | **About 150 min** |

The authors also report **maximum** times of 5.25 minutes for an encode and 0.94 minutes for upscaling and VMAF. Times varied with the video and settings.

**Interpretation:** A source-feature model can avoid the much larger cost of doing many encodes and VMAF tests for a new shot. But feature extraction is not free. Some VIF feature sets are expensive in the Python implementation used by the authors.

**Do not treat these times as universal benchmarks.** They include the authors' hardware and code choices. The paper also warns that CPU time and elapsed wall-clock time are different.

## 10. Strengths of the experiment

These are **critical-reading observations**, not new results reported by the authors.

**The paper compares predictions with an expensive reference.** It does not only ask whether the predicted VMAF numbers look accurate. It checks whether the final ladders are close to those found by exhaustive encoding.

**It uses different kinds of features.** Low-level features, VIF features, and combinations are tested. This helps show which information is useful and whether extra complexity gives a benefit.

**It separates source videos between training and evaluation.** The training, validation, and test sets do not share video titles. This is important because related clips could otherwise make the results look too good.

**It measures computational cost.** A model that is accurate but very slow may not save much work. The paper includes both performance and execution time.

**It tests two useful questions.** Predicting quality at a bitrate and predicting bitrate for a quality target are related but different problems. The study examines both.

## 11. Limits and unanswered questions

These points explain what can and cannot be concluded from the experiment.

### 11.1. Short test sequences

Each sequence contains only **64 frames**. This makes large-scale experiments easier, but it is different from encoding a long shot with many seconds of movement and changing detail.

The experiment does not establish that the same error levels apply to longer sequences.

### 11.2. One encoder and one preset

The reported main experiment uses **libx265 with preset medium**. It does not prove that the same model works without retraining for H.264, AV1, other x265 presets, or other encoder implementations.

### 11.3. One main source resolution

The source clips are prepared as **3840 × 2160**. The authors test many output resolutions, but this is not the same as validating the approach on very different input resolutions and formats.

### 11.4. A chosen VMAF range

The authors focus on **15 ≤ VMAF ≤ 95**. We should not assume the same behavior outside that range, especially near the highest quality levels.

### 11.5. Dataset size and variation

There are **100 sequences**, and reported evaluation uses **30 sequences** outside the training set. The study contains different content, but this is still a limited sample. The large standard deviations show that the gain changes across videos.

### 11.6. VMAF is a model, not a human viewing test

The experiment evaluates visual quality mainly with VMAF. It does not show that human viewers would always make the same decisions, especially for unusual content or artifacts.

### 11.7. The cost of training remains

The prediction method avoids exhaustive tests for a **new** shot after a model has been trained. However, training labels come from many real encodes and quality measurements. That earlier work still has a cost.

### 11.8. The paper does not build synthetic or shortened proxy videos

This study predicts bitrate and quality **directly from source features**. It does not generate artificial frames, build shorter sample videos as substitutes, or test whether such videos behave like the full source when encoded.

This is an important boundary of the paper's findings.

## 12. Main conclusions from this reading

1. **A good encoding decision does not always need a new test encode.** The paper shows that a trained model can make useful choices using features extracted from the source video.
2. **Low-level video features have practical value.** The LLF2 model comes close to the exhaustive reference in the authors' bitrate ladder experiment.
3. **The final decision matters more than one predicted value.** Errors in individual VMAF estimates do not always lead to poor resolution choices.
4. **VIF features help, but they have a cost.** More complex features may improve some results, but the extra processing time must be considered.
5. **The experiment has a clear scope.** Its results concern short, prepared 4K sequences encoded with libx265/medium and evaluated under the chosen VMAF range.
6. **This is evidence for model-based ladder prediction, not for synthetic proxy encoding.** These are different research questions.

### One-sentence takeaway

**In this experiment, a model trained on real encoding results used features from a new source video to choose bitrate and resolution ladders close to those found by many test encodes, but at a much lower prediction cost.**

## 13. Small glossary

| Term | Simple explanation |
|---|---|
| **Shot** | A continuous piece of video without a camera cut. |
| **Bitrate** | Number of bits used per second of video. |
| **Bitrate ladder** | Different bitrate and resolution choices for streaming. |
| **Quality ladder** | Different target quality levels with suitable bitrate and resolution choices. |
| **CRF** | A setting used by some encoders to control quality. |
| **Encoder preset** | A group of settings that changes how an encoder works, often affecting speed and compression efficiency. |
| **VMAF** | A model that estimates video quality by comparing a processed video with a reference. |
| **VIF** | A measure related to the visual information available in an image. |
| **Feature** | A number that describes one property of a video. |
| **Regressor** | A model that predicts a number, such as bitrate or VMAF. |
| **Extra-Trees** | A machine-learning method based on many decision trees. |
| **Cross-over point** | A place where one resolution becomes a better choice than another. |
| **BD-rate** | A method for comparing average bitrate differences between rate–quality curves. |
| **Inference** | Using an already trained model to make a prediction. |

## References (APA 7th edition)

Durbha, K. S., & Bovik, A. C. (2025). Constructing per-shot bitrate ladders using visual information fidelity. *IEEE Transactions on Image Processing, 34*, 7093–7108. https://doi.org/10.1109/TIP.2025.3625750

**Original paper:** [IEEE Xplore](https://ieeexplore.ieee.org/document/11223695/)  
**Open-access version:** [arXiv](https://arxiv.org/abs/2408.01932)  
**Authors' code:** [GitHub](https://github.com/krishnasrikard/Constructing-Per-Shot-Bitrate-Ladders-using-Visual-Information-Fidelity)

---

## AI assistance and document notes

This learning summary and critical reading were prepared with the help of **ChatGPT (OpenAI)** on **October 18, 2026**. It is an original explanation of the cited paper, not a copy of the article. The reported experiment and numbers belong to the paper's authors. The explanations and critical-reading comments are not independently validated research.

For the original scientific work, please read and cite **Durbha and Bovik (2025)**.