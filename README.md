<p align="center">
  <img src="assets/logo.png" alt="Soundstage logo" width="220">
</p>

<h1 align="center">Soundstage</h1>

<p align="center">
  <b>Audio synthesis with context-aware voice inflections &amp; environmental sounds</b><br>
  Final Year Project, BS Computer Science · GIK Institute of Engineering Sciences &amp; Technology · 2023
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-archived%20prototype-lightgrey" alt="Status: archived prototype">
  <img src="https://img.shields.io/badge/Flutter-web-02569B?logo=flutter&logoColor=white" alt="Flutter">
  <img src="https://img.shields.io/badge/Firebase-hosting%20%C2%B7%20auth%20%C2%B7%20storage-FFCA28?logo=firebase&logoColor=black" alt="Firebase">
  <img src="https://img.shields.io/badge/PyTorch-diffusion%20models-EE4C2C?logo=pytorch&logoColor=white" alt="PyTorch">
</p>

<p align="center">
  <a href="docs/Soundstage-Final-Report.pdf"><b>📄 Final report (51 pp.)</b></a> ·
  <a href="docs/Soundstage-Product-Requirements.pdf">Requirements document</a> ·
  <a href="assets/poster.jpg">Poster</a>
</p>

---

> [!NOTE]
> This repository documents the **design, architecture and research** behind Soundstage. The source code is not included: the prototype was a work in progress when the project ended.
>
> The app is no longer functional because its servers have been shut down.

## Contents

- [Overview](#overview)
- [Screenshots](#screenshots)
- [How it works](#how-it-works)
- [System architecture](#system-architecture)
- [Research: diffusion-based audio synthesis](#research-diffusion-based-audio-synthesis)
  - [U-Net building blocks](#u-net-building-blocks)
  - [Text-conditional diffusion generator](#text-conditional-diffusion-generator)
  - [Diffusion vocoder](#diffusion-vocoder)
  - [U-Net vocoder evaluation](#u-net-vocoder-evaluation)
- [Project status](#project-status)
- [Tech stack](#tech-stack)
- [Future work](#future-work)
- [Team](#team)
- [References](#references)

## Overview

Soundstage is a web application that turns a piece of natural text, such as an extract from a novel, into an audio scene. It finds the dialogue in the text, works out who is speaking and what mood the scene is in, and synthesizes each line in a voice that fits the character. In its extended scope it also figures out where the scene is set and generates matching environmental sound (rain, rustling leaves, a flowing river).

Existing text-to-speech tools at the time (Sonantic, Murf.AI, Play.ht) produced high-quality voices but required a person to enter each line, choose a voice and set the emotion by hand. Soundstage's aim was to derive all of that from the text itself.

**Who it was for:** teachers in under-resourced schools who want audiobooks for their classes, visually impaired listeners, independent filmmakers and game developers who can't afford voice actors or a foley artist, and audio platforms producing audiobooks at scale.

## Screenshots

| Landing page | Synthesis tool, Auto mode |
| :---: | :---: |
| <img src="assets/screenshots/landing.jpg" alt="Landing page" width="420"> | <img src="assets/screenshots/tool-auto-mode.jpg" alt="Synthesis tool in auto mode" width="420"> |
| **Manual mode: speech** (age, sex, emotion) | **Manual mode: foley** (rhythmic / non-rhythmic) |
| <img src="assets/screenshots/tool-manual-speech.jpg" alt="Manual speech mode" width="420"> | <img src="assets/screenshots/tool-manual-foley.jpg" alt="Manual foley mode" width="420"> |

<details>
<summary>More screens: About, FAQ, Login</summary>

| About | FAQ | Login |
| :---: | :---: | :---: |
| <img src="assets/screenshots/about.jpg" alt="About page" width="280"> | <img src="assets/screenshots/faq.jpg" alt="FAQ page" width="280"> | <img src="assets/screenshots/login.jpg" alt="Login page" width="280"> |

</details>

## How it works

```mermaid
flowchart LR
    IN["Natural text<br/>e.g. a chapter of a novel"] --> TP

    subgraph TP["Text processing"]
        direction LR
        T1["Dialogue separation<br/>regex over quotation marks"]
        T2["Speaker detection<br/>named-entity recognition"]
        T3["Scene emotion analysis<br/>Limbic"]
        T4["Environment detection *<br/>NER, POS tags, dependency parsing"]
    end

    TP --> AS

    subgraph AS["Audio synthesis"]
        direction LR
        S1["Speech<br/>one voice per character,<br/>inflected by scene emotion"]
        S2["Foley *<br/>text-conditional diffusion"]
    end

    AS --> OUT["Per-line .wav clips<br/>+ full mixed track"]
```

<sub>* Extended scope, see <a href="#project-status">project status</a>.</sub>

1. **Dialogue separation.** Dialogue in prose follows a consistent pattern (text enclosed in quotation marks), so regular expressions extract it reliably and cheaply.
2. **Speaker detection.** Named-entity recognition (NLTK, Stanford CoreNLP, spaCy) identifies who each line belongs to, so every character keeps a distinct, recognizable voice. DistilBERT models and a separate NER pipeline were also tried.
3. **Scene emotion analysis.** Emotion is computed once per scene with the Limbic library, not per line, which keeps compute cost manageable while still giving voices the right tone.
4. **Environment detection.** NER, part-of-speech tagging and dependency parsing (spaCy, NLTK) pick out the setting, such as a forest or a seaside, to drive foley generation.

Users can let the app do all of this (**Auto mode**) or set each character's age, sex and emotion themselves (**Manual mode**).

## System architecture

The report describes the system with the **4+1 architectural view model**. Physically, the front end and data live on Firebase, and all heavy computation runs on a separate processing server.

```mermaid
flowchart TB
    USER(["User's browser"])

    subgraph FB["Google Firebase"]
        AUTH["Authentication"]
        HOST["Hosting<br/>Flutter web app"]
        DB[("Firestore<br/>per-user history")]
        BUCKET[("Cloud Storage<br/>generated audio")]
    end

    subgraph GCP["Google Cloud VM"]
        API["Flask HTTP API"] --> NLP["Text processor"] --> SYN["Synthesis pipeline<br/>speech, foley *"]
    end

    USER -- "sign in" --> AUTH
    USER -- HTTPS --> HOST
    HOST -- "POST text + user id" --> API
    SYN -- "clip metadata" --> DB
    SYN -- ".wav clips" --> BUCKET
    DB -- "history" --> HOST
    BUCKET -. "playback" .-> USER
```

### Request lifecycle

Synthesis can take minutes, so the API replies straight away with a job ID and does the work on a background thread. The web app picks up the results from Firestore as each clip is written.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Web as Flutter web app
    participant API as Flask API
    participant NLP as Text processor
    participant TTS as Speech synthesis
    participant FB as Firebase

    User->>Web: Paste text, pick Auto or Manual mode
    Web->>API: POST text and user id
    API->>FB: Create history entry (job id, time, text)
    API-->>Web: Job id
    Note over API: Remaining work runs on a background thread
    API->>NLP: Extract quotes, speakers and cues
    NLP-->>API: Table of speaker, quote, gender
    loop Each line of dialogue
        API->>TTS: Synthesize line in the speaker's voice
        TTS-->>API: Audio clip
        API->>FB: Upload clip and add metadata
    end
    API->>FB: Upload full mixed track
    Web->>FB: Read history
    FB-->>Web: Clips ready to play
```

### UML views from the report

<details>
<summary><b>Use case view</b>: register/login, input text, select mode, select characteristics, generate audio</summary>
<br>
<img src="assets/diagrams/use-case.png" alt="Use case diagram" width="560">
</details>

<details>
<summary><b>Process view</b>: activity diagram of the Auto and Manual flows</summary>
<br>
<img src="assets/diagrams/activity.png" alt="Activity diagram">
</details>

<details>
<summary><b>Physical view</b>: deployment diagram</summary>
<br>
<img src="assets/diagrams/deployment.png" alt="Deployment diagram">
</details>

<details>
<summary><b>Development view</b>: three components</summary>
<br>

```mermaid
flowchart LR
    UI["«component»<br/>User Interface"]
    TP["«component»<br/>Text Processor"]
    SP["«component»<br/>Synthesis Pipeline"]
    UI -- "natural text" --> TP
    TP -- "normalized text" --> SP
    UI -- "synthesis details" --> SP
    SP -- "synthesized audio" --> UI
```

</details>

## Research: diffusion-based audio synthesis

Denoising diffusion, the approach behind image generators like DALL·E 2 and Midjourney, was the core research direction. Images and audio are different kinds of data, so the pipeline converts audio to and from (mel) spectrograms and uses **1D U-Nets** as the denoiser. The work builds on Flavio Schneider's *ArchiSound* [[3]](#references) and its open-source `a-unet` library.

### U-Net building blocks

A U-Net is an encoder–decoder network with skip connections that preserve fine detail at every resolution. Here it is assembled recursively from **blocks**:

```mermaid
flowchart LR
    X(["input"]) --> DOWN["Downsample"] --> IT1["Items"] --> INNER["Inner block<br/>(recursive)"] --> IT2["Items<br/>(mirrored)"] --> UP["Upsample"] --> SKIP["Skip merge"] --> Y(["output"])
    X -. "skip connection" .-> SKIP
```

Each block runs a stack of **items**, the processing units at that resolution:

| Item | Code | What it does |
| --- | :---: | --- |
| ResnetItem | **R** | Convolutional processing |
| ModulationItem | **M** | Modulates channels from a feature vector (e.g. the diffusion time step) |
| AttentionItem | **A** | Self-attention between region vectors |
| CrossAttentionItem | **C** | Cross-attention to external embeddings (e.g. text) |
| FeedForwardItem | **F** | MLP-style processing of region vectors |
| InjectItem | **I** | Injects external channels (e.g. a conditioning signal) |

Conditioning can enter through feature vectors, embeddings or extra channels. **Plugins** such as time conditioning and classifier-free guidance wrap the U-Net without changing the network itself.

### Text-conditional diffusion generator

**Why text conditioning.** Frozen pretrained language models already steer image diffusion toward a text prompt, so the same should work for audio. For Soundstage, that means describing a sound effect in words ("rain on a tin roof") and generating it. The same model could later produce background music too.

**Method.** A frozen **T5** transformer encodes the description into an embedding. The U-Net attends to it through cross-attention blocks at several resolutions. Training clips have a fixed length, so each description gets a chunk tag (e.g. *"1 of 4"*) to say which part of a longer clip the model is looking at.

```mermaid
flowchart LR
    subgraph TRAIN["Training"]
        direction LR
        D1["Text description<br/>+ chunk tag"] --> T5A["Frozen T5"] --> E1["Embedding"]
        A1["Audio clip"] --> N1["Add noise"] --> U1["1D U-Net<br/>with cross-attention"] --> L1["Diffusion loss"]
        E1 --> U1
    end
```

```mermaid
flowchart LR
    subgraph INFER["Inference"]
        direction LR
        D2["Text description"] --> T5B["Frozen T5"] --> E2["Embedding"]
        Z["Pure noise"] --> UA["U-Net"] --> UB["U-Net"] --> UC["U-Net"] --> OUT2["Generated audio"]
        E2 --> UA
        E2 --> UB
        E2 --> UC
    end
```

| Training setup (final presentation) | |
| --- | --- |
| Optimizer | AdamW |
| Steps | 1M |
| Batch size | 32 |
| Data | Audiobooks |
| Clip size | 2²¹ samples (~44 s at 48 kHz) |

**Result.** Text conditioning worked well: generated audio matched its description. Used directly as a text-to-speech model, however, the output was **mostly unintelligible**. Foley was in the extended scope, so this component was not taken further.

### Diffusion vocoder

**Why.** Mel spectrograms weight frequencies the way human hearing does, which makes them a good intermediate representation. Turning a spectrogram back into a waveform without losing detail is hard, though. Iterative methods like Griffin-Lim leave audible artifacts. ArchiSound proposes a small change that turns the same U-Net into a high-quality **vocoder**.

**Method.** The mel spectrogram is *flattened* back to waveform length with a learned **transposed 1D convolution**. That flattened signal is fed to the U-Net as extra channels next to the noisy waveform being denoised.

```mermaid
flowchart LR
    subgraph VTRAIN["Vocoder training"]
        direction LR
        W["Waveform"] --> MEL["Mel transform"] --> SPEC["Mel spectrogram"] --> CT["ConvTranspose1d"] --> FLAT["Flat mel"]
        W --> NZ["Add noise"] --> CAT["Concatenate channels"]
        FLAT --> CAT --> UV["1D U-Net"] --> LV["Diffusion loss"]
    end
```

```mermaid
flowchart LR
    subgraph VINFER["Vocoder inference"]
        direction LR
        SPEC2["Mel spectrogram"] --> CT2["ConvTranspose1d"] --> FLAT2["Flat mel"]
        Z2["Noise"] --> V1["U-Net"] --> V2["U-Net"] --> V3["..."] --> WAV["Waveform"]
        FLAT2 --> V1
        FLAT2 --> V2
    end
```

The key is lining the transposed convolution up with the STFT that produced the spectrogram. An STFT is effectively a 1D convolution with long sine and cosine windows, followed by a magnitude and a mel projection. Neither of those last two steps changes the time axis. So if the convolution mirrors the STFT's geometry, each output region covers exactly the stretch of waveform its spectrogram frame came from:

| STFT parameter | Transposed-convolution parameter |
| --- | --- |
| Window length | Kernel size |
| Hop length | Stride |
| Padding / centering | Matching padding |

### U-Net vocoder evaluation

- **High-quality waveforms.** The diffusion vocoder reconstructed high-fidelity audio from mel spectrograms, *as long as* the U-Net has enough convolutional (ResNet) blocks at its first, highest-resolution stage. Those blocks are what recover the high frequencies.
- **Capacity scales with mel channels (hypothesis).** Following ArchiSound's author, more mel channels should improve quality for two reasons. The spectrogram discards less information, and the transposed convolution gets more channels to flatten, so it has more capacity.
- **Text-conditional TTS fell short.** The same diffusion backbone matched text descriptions well for general audio but did not produce intelligible speech.

The evaluation was qualitative. No objective metrics were reported. Full details are in chapters 4–5 of the [final report](docs/Soundstage-Final-Report.pdf).

## Project status

The base scope (text processing, 1–2 voice models, automatic synthesis, integrated web app) was delivered. The extended scope was still in development when the project ended.

| Component | Status (May 2023) |
| --- | --- |
| Web app: Flutter UI, Firebase auth, history, playback | ✅ Done |
| Integration: web app ↔ Flask API ↔ Firebase | ✅ Done |
| Voice synthesis | ✅ Done |
| Text analysis: dialogue separation, speaker detection | 🟡 Partial: speakers are found only when attribution follows a standard structure (e.g. *"…," said Tom*) |
| Diffusion vocoder | 🔬 Research prototype, high-quality reconstructions |
| Text-conditional diffusion | 🔬 Research prototype, good for described sounds, unintelligible as TTS |
| Foley synthesis * | 🚧 In progress (extended scope) |
| 3–4 voice models, environment detection * | 🚧 Extended scope, not implemented |

**Known limitations:** English-only input; varied writing styles can confuse text processing; emotion in speech is not always as intended; some words are mispronounced because of dataset gaps; scaling currently means redeploying the whole server.

## Tech stack

| Layer | Tools |
| --- | --- |
| Design | Figma |
| Front end | Flutter (Dart), Roboto |
| Hosting & data | Firebase Hosting, Authentication, Firestore, Cloud Storage |
| Back end | Python, Flask on a Google Cloud VM |
| NLP | spaCy, textacy, NLTK, Stanford CoreNLP, Limbic, pandas |
| Audio ML | PyTorch, TensorFlow, `a-unet` (ArchiSound), T5 text encoder |
| Tested on | Chrome, Firefox, Edge |

## Future work

1. **Better text processing.** Handle complex dialogue attribution and finer-grained detail.
2. **More control.** Let users specify emotion and other generation parameters explicitly.
3. **More models.** Add voices, accents and languages, plus background music generation.
4. **Foley generation.** Finish text-conditional sound-effect synthesis.
5. **Timeline editor.** Trim and arrange generated clips in the app.
6. **Scalability.** Move beyond redundant whole-server deployments.

## Team

| | |
| --- | --- |
| **Students** | Asad Shahab · Shams Ghani · Syed Nazim Shah · Maida Shafaq |
| **Advisor** | Dr. Zahid Halim |
| **Co-advisor** | Engr. Ahsan Shah |
| **Institution** | Faculty of Computer Science & Engineering, Ghulam Ishaq Khan Institute of Engineering Sciences & Technology |

<details>
<summary>Poster</summary>
<br>
<img src="assets/poster.jpg" alt="Soundstage project poster" width="600">
</details>

## References

1. Z. Kong, W. Ping, J. Huang, K. Zhao, B. Catanzaro. *DiffWave: A Versatile Diffusion Model for Audio Synthesis.* 2021. [arXiv:2009.09761](https://arxiv.org/abs/2009.09761)
2. J. Shen et al. *Natural TTS Synthesis by Conditioning WaveNet on Mel Spectrogram Predictions* (Tacotron 2). 2018. [arXiv:1712.05884](https://arxiv.org/abs/1712.05884)
3. F. Schneider. *ArchiSound: Audio Generation with Diffusion.* 2023. [arXiv:2301.13267](https://arxiv.org/abs/2301.13267)
4. C. D. Manning et al. *The Stanford CoreNLP Natural Language Processing Toolkit.* 2014. [PDF](https://nlp.stanford.edu/pubs/StanfordCoreNlp2014.pdf)

---

<sub>© 2023 the Soundstage team. All rights reserved. Figures in this README were redrawn for this repository. Diagram images and screenshots are from the team's own report.</sub>
