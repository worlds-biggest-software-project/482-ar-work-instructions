# AR Work Instructions

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source platform that renders step-by-step maintenance, assembly, and safety procedures directly onto physical equipment through augmented reality, eliminating paper manuals and reducing worker error rates.

AR Work Instructions is an AR-native guidance system for industrial workplaces. It targets process engineers, maintenance technicians, and plant operators who currently rely on paper manuals, PDFs, and static videos to perform complex assembly, inspection, and repair tasks. By spatially anchoring each instruction step to the physical equipment in a worker's field of view, it removes the context-switching that causes errors, missed steps, and slow onboarding.

---

## Why AR Work Instructions?

- **No open-source alternative exists.** Every comparable platform (PTC Vuforia, Scope AR, Augmentir, Microsoft Dynamics 365 Guides, Taqtile Manifest) is fully proprietary with opaque, enterprise-only pricing. Small and mid-size manufacturers are effectively locked out.
- **Vendor lock-in is pervasive.** There is no open standard for exporting procedure content between competing platforms. Investing in one vendor's authoring tools means rewriting everything if you switch.
- **Pricing excludes SMEs.** PTC Vuforia requires enterprise quotes; Microsoft Dynamics 365 Guides runs approximately $65/user/month and is tied to the Dynamics 365 ecosystem; most others do not publish pricing at all.
- **AI capabilities are fragmented.** PTC offers AI visual inspection (Step Check) as a separate add-on. Augmentir has AI content conversion but bolted AR on as a late addition (September 2025). No single platform delivers AI-powered authoring, inspection, and adaptive procedures in a unified open stack.
- **Hardware lock-in persists.** Microsoft Guides requires HoloLens 2 (hardware no longer in active development). Siemens relies on TeamViewer Spatial Editor for AR delivery. An open platform built on WebXR and WebAR can target any device.

---

## Key Features

### Procedure Authoring

- Web-based no-code WYSIWYG editor for creating step-by-step instructions with text, images, video, and 3D models
- Guided wizard for common procedure types to reduce authoring cycle time
- AI-powered SOP-to-procedure conversion: upload a PDF or Word document and generate a structured draft procedure
- glTF 2.0 model import with Draco mesh compression for step-level 3D overlays
- Procedure versioning with review, approval workflow, and instant rollback

### AR Playback & Spatial Anchoring

- Mobile AR playback on iOS and Android via WebAR or lightweight native app
- QR code and marker-based spatial anchoring for reliable equipment identification
- Computer-vision object detection for marker-free environments with fallback to fiducial markers
- Headset support for HoloLens 2 and Meta Quest via WebXR
- Lightweight WebAR delivery without native app installation

### Workforce & Compliance

- Completion tracking with per-step timestamps, operator ID, and pass/fail records for audit
- Role-based access control: author, reviewer/approver, and operator roles
- Multi-language rendering at runtime without maintaining separate procedure files
- Skills-based procedure assignment matching procedure complexity to operator competency

### Enterprise Integration

- MES/ERP webhook integration (SAP, Oracle) to auto-assign procedures to the correct workstation and shift
- Offline-first architecture with local procedure cache and sync-on-reconnect
- Remote expert video call with AR annotation via WebRTC
- CAD system connectors for direct geometry import from SolidWorks, Siemens NX, and PTC Creo

### AI-Powered Quality

- AI visual inspection using device camera for pass/fail step checks
- Adaptive procedures that reorder or skip steps based on worker skill level and real-time context
- Natural language query against procedure libraries
- Anomaly detection in completion data to surface hidden process failures

---

## AI-Native Advantage

Current platforms treat AI as an add-on: PTC's Step Check is a separate product, Augmentir's AR module was grafted onto an existing connected-worker platform, and Tulip's AI Composer handles only SOP conversion. An AI-native approach unifies these capabilities from the ground up. Generative AI converts unstructured expert video recordings into structured procedures by auto-captioning and auto-segmenting steps. Computer vision validates each completed step in real time, detecting missing or misaligned parts without a separate inspection tool. The system adapts procedure detail and step ordering to each worker's assessed skill level, and anomaly detection across completion data surfaces process drift before it causes defects.

---

## Tech Stack & Deployment

The platform targets self-hosted, cloud, and hybrid deployment modes, including air-gapped on-premises installations for defence and regulated environments. Core technology choices are built on open standards: glTF 2.0 (Khronos Group) for 3D asset delivery, WebXR (W3C) for cross-device AR rendering, and WebRTC (W3C/IETF) for sub-300 ms remote expert video. Spatial anchor persistence uses depth cameras, IMUs, and pre-scanned point-cloud maps for robustness in industrial environments with reflective surfaces and low-texture floors. CAD integration supports open-format intermediaries (STEP, glTF) alongside vendor-licensed SDK paths for native formats.

---

## Market Context

Augmented reality manufacturing software is projected to exceed $700 million in value by 2026, with approximately 28 million AR and mixed-reality smart glasses expected to ship in 2026. Enterprise adoption is accelerating: Toyota's HoloLens 2 deployment cut training times by 50%, and pharmaceutical manufacturer Korber's AR deployment helped operators master new tasks 44% faster. The primary buyers are process engineers, operations directors, and training managers at discrete manufacturers, aerospace MRO providers, pharmaceutical producers, and utility companies, driven by labour shortages, increasing product complexity, and regulatory pressure to document every step of regulated processes.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
