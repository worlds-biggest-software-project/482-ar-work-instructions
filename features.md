# AR Work Instructions — Feature & Functionality Survey

> Candidate #482 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| PTC Vuforia Expert Capture | Commercial SaaS | Proprietary, quote-based | https://www.ptc.com/en/products/vuforia/vuforia-expert-capture |
| Scope AR WorkLink | Commercial SaaS | Proprietary, enterprise licence | https://www.scopear.com/product |
| Augmentir | Commercial SaaS | Proprietary, subscription | https://www.augmentir.com/ |
| Frontline.io | Commercial SaaS | Proprietary, subscription | https://www.frontline.io/ |
| Plutomen | Commercial SaaS | Proprietary, subscription | https://pluto-men.com/ |
| Siemens Teamcenter Easy Plan | Commercial (PLM-bundled) | Proprietary, enterprise licence | https://www.siemens.com/en-us/products/tecnomatix/offerings/teamcenter-easy-plan/ |
| Microsoft Dynamics 365 Guides | Commercial SaaS | Proprietary, per-user/month | https://learn.microsoft.com/en-us/dynamics365/mixed-reality/guides/ |
| Taqtile Manifest | Commercial SaaS | Proprietary, quote-based | https://taqtile.com/manifest/ |
| Proceedix (SymphonyAI) | Commercial SaaS | Proprietary, subscription | https://www.proceedix.com/ |
| Librestream Onsight | Commercial SaaS | Proprietary, enterprise licence | https://librestream.com/platform/ |
| Tulip | Commercial SaaS | Proprietary, subscription | https://tulip.co/ |

---

## Feature Analysis by Solution

### PTC Vuforia Expert Capture

**Core features**
- Step-by-step AR work instruction authoring and delivery via the Vuforia Vantage delivery app
- Mobile Capture: real-time procedure capture and documentation using existing smartphones and tablets
- Step Check: AI-powered computer-vision visual inspection with pass/fail alerts in AR
- 3D CAD import for embedding native geometry in procedures
- Vuforia Insights analytics dashboard: tracks procedure execution, quality issues, and usage by worker
- Distribution Centre for managing permissions and distribution lists
- Collaborative, web-based SaaS authoring interface
- Integration with PTC ThingWorx IIoT platform

**Differentiating features**
- Step Check AI inspection is one of the earliest AI-powered error-detection features in AR work instructions; detects missing or misaligned parts automatically
- Deep integration with PTC Creo and Windchill PLM for CAD-to-instruction pipelines
- Mobile Capture lowers barrier for procedure creation — no headset required at authoring stage

**UX patterns**
- Authoring on PC web; playback on Vuforia Vantage (mobile/headset)
- Guided wizard approach to procedure creation for non-technical authors
- AR overlays are spatially anchored to physical assets; tethered instruction cards follow equipment location

**Integration points**
- PTC ThingWorx (IIoT)
- PTC Windchill (PLM)
- REST API for enterprise system connectors
- SAP and Oracle ERP via ThingWorx integration layer

**Known gaps**
- No native offline-first architecture; connectivity drops affect playback
- High per-user pricing puts it out of reach for SMEs
- Limited support for non-PTC CAD formats without conversion
- Remote expert video collaboration requires Vuforia Chalk as a separate product

**Licence / IP notes**
- Fully proprietary. Pricing not published; enterprise quotes required. Vuforia Engine SDK has a free development tier with watermarking; production use requires paid licence.

---

### Scope AR WorkLink

**Core features**
- Combined AR work instructions and real-time remote AR assistance in a single platform
- No-code, web-based content creation with AI-assisted authoring interface
- Native CAD import (publishes across Windows, iOS, Android without conversion)
- Cross-platform delivery: mobile, headset (HoloLens 2, Apple Vision Pro, Quest), and desktop
- Real-time collaborative interaction with 3D content during remote sessions
- MES and ERP integration
- Session naming with work order details (v2.21.0) to support shift handover

**Differentiating features**
- First platform to combine work instructions and remote expert assistance natively in one product, eliminating the need for two separate tools
- Apple Vision Pro support via visionOS — cross-platform content reuse without re-authoring
- Demonstrated 30–40% reduction in task time and zero recorded errors in MRO deployments

**UX patterns**
- Spatial computing instructions with 3D overlay directly on equipment
- Progressive workflow gating: workers cannot advance until step conditions are satisfied
- Session resume capability so multiple workers can hand off a procedure mid-task

**Integration points**
- MES and ERP APIs (SAP, Oracle)
- REST and webhook connectors
- Compatible with HoloLens 2, Apple Vision Pro, Quest 3, iOS, Android, Windows

**Known gaps**
- Authoring still requires moderately technical users for complex 3D content
- Analytics dashboards less mature than dedicated analytics platforms
- Pricing opaque; enterprise-only packaging limits SME access

**Licence / IP notes**
- Proprietary commercial SaaS. No open-source components published. Enterprise licence with per-user or site-based pricing (not publicly disclosed).

---

### Augmentir

**Core features**
- AI-native connected worker platform (agentic AI + GenAI via the "Augie" suite)
- Content transformation: converts existing SOPs, training videos, and PDFs into interactive step-by-step procedures automatically
- Skills management: digitises skills matrix, identifies skill gaps, enables targeted upskilling assignments
- Industrial AI Agent Studio: no-code builder for custom AI agents (5 Why Coach, Root Cause Investigator, Data Analyst)
- AR add-on (launched September 2025): immersive AR onboarding and training overlay on real equipment
- True Productivity and True Performance metrics for workforce performance visibility
- Procedure authoring and execution on mobile, tablet, and wearables

**Differentiating features**
- Only platform to integrate AI-generated skills gap analysis directly with work instruction assignment
- Augie GenAI suite auto-converts legacy content (video, PDF, SOP) into procedures — uniquely reduces migration effort
- Agentic AI agents that can autonomously investigate root causes and coach workers through corrective actions
- Named Frost & Sullivan #1 innovation leader in Augmented Connected Worker category (January 2026)

**UX patterns**
- Hire-to-retire worker journey framing: onboarding, daily work, upskilling, performance review in one platform
- Conversational AI assistant (Augie) embedded in procedure execution for real-time help
- Personalised procedure assignment based on skills profile

**Integration points**
- SAP, Oracle, and custom ERP/MES via REST API
- LMS integrations for training record export
- AR wearable support via September 2025 AR extension

**Known gaps**
- AR capabilities are a recent add-on (2025), not native to the platform — spatial anchoring less mature than dedicated AR-first tools
- Heavy AI dependency may create trust barriers in safety-critical regulated environments
- Platform complexity may overwhelm smaller teams seeking only a simple procedure player

**Licence / IP notes**
- Proprietary SaaS. Pricing not published; module-based subscription model. No open-source components.

---

### Frontline.io

**Core features**
- Interactive step-by-step 3D flows for training, support, and troubleshooting
- Visual Workflow Editor: drag-and-drop procedure builder
- Fast Track Pro: CAD file import to digital twin without leaving the platform
- Multi-device support: AR/VR/MR headsets, PC, tablet, mobile
- Localization support (preferred language and regional format rendering)
- AI Guidance for resource access and workflow navigation (v25.1)
- Interactive Flows: clickable links within step descriptions
- Analytics: flow run recording and usage dashboards

**Differentiating features**
- Fast Track Pro CAD-to-digital-twin pipeline is platform-native without needing PLM connectors
- Broad headset compatibility: HoloLens 2, Quest 3, Vuzix Ultralight, RayNeo X3, DigiLens, HMS SINGRAY G2
- v25.1 (2025) added built-in localization, removing the need for separate translation management

**UX patterns**
- Single authoring environment for AR, VR, PC, and mobile — single source of truth across device types
- 3D step visualizations reduce reliance on text-heavy procedures

**Integration points**
- Compatible with major AR/VR/MR headsets across vendors
- No published MES/ERP connector list; integration appears primarily device-facing

**Known gaps**
- ERP/MES integration depth not prominently documented; likely less mature than PTC/Scope AR
- Remote expert video assistance not listed as a core feature
- Analytics appears basic compared to dedicated business intelligence tools

**Licence / IP notes**
- Proprietary SaaS. Subscription pricing; specific tiers not publicly disclosed.

---

### Plutomen

**Core features**
- Three-module suite: Connect (AR remote assistance), Workflow (digital work instructions), Assist (immersive AR/VR training)
- QR code and search-based procedure access for on-the-job learning
- No-code editor for creating immersive 3D AR/VR training experiences
- Chat, live video, audio, AR annotation, and file sharing in remote assistance sessions
- Compatible with RealWear, Vuzix, Oculus/Meta Quest, Microsoft HoloLens
- Paperless procedures with mobile and wearable playback

**Differentiating features**
- Tight vertical integration between remote assistance, work instructions, and immersive training in one pricing bundle
- GEDC 2025 Startup Innovation Award — recognised for SME-friendly pricing and emerging-market reach

**UX patterns**
- QR code scan as the primary job-site access point — no search required in most deployments
- Workers access guidance through familiar mobile devices before graduating to AR headsets

**Integration points**
- ERP/MES connectors available; specific systems not publicly enumerated
- AR/VR wearable ecosystem integrations

**Known gaps**
- AI capabilities less developed than Augmentir or PTC Step Check
- Spatial anchoring and 3D CAD fidelity less mature than PTC/Scope AR
- Analytics and compliance reporting functionality not prominently documented
- Documentation in English only in early versions; localization coverage unclear

**Licence / IP notes**
- Proprietary SaaS. India-headquartered; pricing competitive for emerging markets but enterprise tiers available. No open-source components.

---

### Siemens Teamcenter Easy Plan

**Core features**
- PLM-native work instruction authoring integrated with MBOM and process planning
- AR work instructions via TeamViewer Spatial Editor integration
- AI-powered translation of work instructions (weeks reduced to hours; available from 2506 release)
- Real-time synchronisation of instructions with engineering changes (ETO production)
- Line balancing, process planning, and MBOM management in the same tool
- Enterprise Recipe Management for regulated manufacturing

**Differentiating features**
- Only tool where work instructions are authored directly from the MBOM within a full PLM context — change propagation is automatic
- AI translation preserves technical accuracy and context across languages — unique differentiator for multinational OEMs
- ETO (Engineer-to-Order) real-time sync addresses a gap no other AR-first tool covers

**UX patterns**
- Desktop-first PLM workflow; AR delivery via separate Spatial Editor viewer
- Change management UI familiar to engineering change boards

**Integration points**
- Native Siemens NX CAD integration
- SAP and Oracle ERP via Teamcenter connectors
- TeamViewer Spatial Editor for AR delivery
- MES integration via Opcenter

**Known gaps**
- AR delivery depends on TeamViewer Spatial Editor — a third-party integration, not native AR
- Primarily suited to large OEMs with existing Teamcenter licences; high cost of entry for SMEs
- AR playback UX less intuitive than AR-native tools
- No real-time remote expert assistance in the core product

**Licence / IP notes**
- Proprietary, part of Siemens Xcelerator portfolio. Enterprise pricing; bundled with Teamcenter licences. No open-source components.

---

### Microsoft Dynamics 365 Guides

**Core features**
- HoloLens 2-native holographic work instruction authoring and playback
- PC authoring app + HoloLens placement app (two-step authoring workflow)
- Gaze-controlled navigation: no hands required to advance steps
- Instruction cards with images, videos, and 3D holograms spatially tethered to equipment
- Real-time remote collaboration via Microsoft Teams (desktop/mobile)
- Power BI dashboards for procedure analytics
- Integration with Dynamics 365 Field Service and Finance & Operations

**Differentiating features**
- Deep Microsoft ecosystem integration: Teams, Power BI, Dataverse, Field Service — no middleware required
- Gaze-only navigation is uniquely hands-free even without voice commands
- Available as a standalone D365 module, making procurement straightforward for existing Microsoft customers

**UX patterns**
- Spatial tethering of instruction cards to real-world positions — cards follow the physical object as the worker moves
- Familiar Microsoft enterprise UX lowers training overhead for IT administrators

**Integration points**
- Microsoft Teams (live remote collaboration)
- Power BI (analytics)
- Dynamics 365 Field Service (work order integration)
- Dataverse (data model)
- Azure Spatial Anchors (cloud anchor persistence)

**Known gaps**
- HoloLens 2 hardware dependency — no mobile AR or alternative headset support
- HoloLens 2 is no longer in active development (hardware EOL signals); long-term platform viability uncertainty
- Limited CAD import options compared to Scope AR or Vuforia
- No AI-powered content generation or visual inspection features

**Licence / IP notes**
- Proprietary Microsoft product. Per-user/month subscription pricing publicly available (~$65/user/month at last published rate, subject to change). Requires Dynamics 365 ecosystem.

---

### Taqtile Manifest

**Core features**
- Digital work instruction authoring with AR overlay and spatial anchoring to physical equipment or digital twins
- Supports iPad/mobile, AR headsets, and desktop scaling in a single platform
- Air-gapped and on-premises deployment options (hosted cloud, private cloud, on-prem, edge, unclassified/classified)
- Enterprise-grade access control, encryption, and location-level data capture controls
- Live sensor data visualisation integrated into procedure steps
- Rules-based equipment telemetry alerts embedded in work instructions
- Remote collaboration and assistance with spatial computing annotations
- Knowledge capture from domain experts

**Differentiating features**
- Only AR work instructions platform certified for defence/military air-gapped deployments — no cloud dependency required
- Live IoT sensor data embedded directly in AR instruction steps (e.g., show current torque vs. target)
- MRO aerospace focus: partnered with FTAI Aviation (MRO Americas 2025) and Delta Black Aerospace
- DigiSaaS partnership with DigiLens for embedded headset distribution channel

**UX patterns**
- Procedure scaling from tablet to AR headset with the same content — operators choose device, not content owners
- Digitised manuals with AR enhancement rather than full 3D model dependency

**Integration points**
- IoT sensor data integration (proprietary connectors)
- Microsoft HoloLens certified
- DigiLens headsets
- Defence systems integration (air-gapped)

**Known gaps**
- Primarily aerospace/defence vertical — limited out-of-box templates for discrete manufacturing or utilities
- AI content generation not prominently featured (early-stage relative to Augmentir)
- Pricing not publicly available; likely high for non-defence use cases

**Licence / IP notes**
- Proprietary SaaS with on-prem option. Quote-based enterprise pricing. Designed for ITAR/CMMC-regulated environments.

---

### Proceedix (SymphonyAI)

**Core features**
- SaaS-based digitisation of procedures, work instructions, and inspections
- Interactive digital work instructions with embedded videos, images, and diagrams
- Conditional/adaptive workflows: additional steps triggered by real-time observed conditions
- Offline-first architecture: procedure execution without connectivity; sync on reconnection
- Open API for ERP, MES, and enterprise system integration
- Mobile execution via smartphones, tablets, and smart glasses (wearable-first)
- Digital inspections with data capture and compliance record generation
- Integration with SymphonyAI industrial AI platform

**Differentiating features**
- Offline-first is a core architectural pillar, not an afterthought — reliable in remote/underground plant environments
- Conditional workflow branching based on field observations (e.g., abnormal inspection finding triggers additional steps automatically)
- Backed by SymphonyAI's industrial AI platform for predictive maintenance and process optimisation cross-sell

**UX patterns**
- Hands-free on smart glasses; touch on tablets; responsive to environment conditions in real time
- Compliance-first design: every execution creates an auditable record for regulatory purposes

**Integration points**
- Open API (REST) for ERP and MES
- Smart glasses compatibility (RealWear, Vuzix)
- SymphonyAI platform integrations

**Known gaps**
- AR 3D spatial overlay capabilities appear less advanced than headset-native tools
- Content authoring tools less mature for complex 3D assets
- Less brand recognition than PTC or Microsoft in North American market

**Licence / IP notes**
- Proprietary SaaS owned by SymphonyAI (AI investment holding company). Subscription pricing not publicly disclosed. No open-source components.

---

### Librestream Onsight

**Core features**
- AI-powered AR platform combining remote expert assistance, AR-enhanced diagnostics, and AI-guided workflows
- Live translation in 27 languages during remote sessions
- Object recognition for equipment identification without QR codes
- Sensor/IoT data integration and visualisation during sessions
- Hands-free support via wearables (RealWear, Vuzix, and others)
- Operates in extreme low-bandwidth conditions (cellular, satellite)
- Onsight NOW: secure video collaboration for defence/sustainment (Lockheed Martin + Microsoft partnership, July 2025)
- Deployed globally across 193 countries on 6 continents

**Differentiating features**
- Live AI translation during remote expert sessions — unique for multinational field operations
- Extreme low-bandwidth optimisation makes it viable for remote/offshore/field environments where other platforms fail
- Defence partnership with Lockheed Martin for secure field collaboration (Onsight NOW)

**UX patterns**
- Remote expert controls field camera remotely to redirect the worker's view
- Context-aware AI guidance surfaced at the moment of need without interrupting workflow

**Integration points**
- Enterprise wearables (RealWear, Vuzix, ECOM Instruments)
- REST API for enterprise system connectors
- Microsoft teams of integrations for Onsight NOW

**Known gaps**
- Primarily a remote assistance tool; standalone AR work instruction authoring less mature than Vuforia or Scope AR
- Procedure authoring tools not as prominent in marketing materials
- Free plan limited; enterprise pricing not disclosed

**Licence / IP notes**
- Proprietary commercial SaaS. Canadian company. Enterprise pricing. No open-source components.

---

### Tulip

**Core features**
- No-code app builder for frontline operations with AR capability
- AI Composer: transforms existing SOPs, documents, and work instructions into interactive Tulip apps (cuts build time by up to 80%)
- Digital work instructions with operator guidance, quality gates, and material picking verification
- Poka-yoke workflows to eliminate quality defects
- Auto-provisioning of stations and interfaces on first operator login
- Broad sensor, machine, and IoT device integration
- Real-time visibility dashboard for supervisors
- Platform Release 324 (June 2025): enhanced analytics and integration capabilities

**Differentiating features**
- No-code app builder positions it as the most accessible authoring platform for non-IT process engineers
- AI Composer is one of the most capable SOP-to-digital-procedure automation tools available
- Broader operations platform scope: includes quality, inventory, and compliance beyond work instructions

**UX patterns**
- App-based framing: authors build Tulip "apps" rather than "procedures" — more flexible but more complex
- Operator-facing interfaces optimised for shop-floor touchscreens and tablets
- Continuous improvement loop: operators can flag issues in-step, feeding back to process engineers

**Integration points**
- Machine and sensor connectors (OPC-UA, MQTT, Modbus)
- SAP and other ERP via REST API
- Quality management and MES system integrations

**Known gaps**
- AR spatial overlay capabilities are secondary to the no-code operations platform core
- Not headset-native; AR features less mature than dedicated AR-first tools
- Requires technical authoring effort despite no-code positioning for complex procedures

**Licence / IP notes**
- Proprietary SaaS. Subscription pricing available on request. No open-source core. Significant VC backing.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Step-by-step sequential procedure player with progress tracking
- Media-rich steps: images, video clips, and 3D model overlays
- Web-based WYSIWYG procedure authoring (no 3D modelling skills required)
- Multi-device delivery: at minimum mobile and tablet; headset support increasingly expected
- Completion tracking with timestamped execution records for audit/compliance
- Offline procedure playback with sync-on-reconnect
- Role-based access control for authors, reviewers, and operators

### Differentiating Features
- AI-powered visual inspection (pass/fail detection) — currently unique to PTC Step Check; becoming table stakes
- Automatic SOP/document-to-procedure conversion via GenAI (Augmentir Augie, Tulip AI Composer)
- Native PLM/CAD change propagation to procedures (Siemens Teamcenter only)
- Air-gapped / on-premises deployment with no cloud dependency (Taqtile Manifest)
- Live IoT sensor data embedded in AR instruction steps (Taqtile)
- Extreme low-bandwidth remote collaboration (Librestream Onsight)
- Real-time live AI translation across 27+ languages during remote sessions (Onsight)
- Skills matrix integration: procedure assignment driven by assessed competencies (Augmentir)

### Underserved Areas / Opportunities
- True open-source AR work instructions platform — no credible open-source alternative exists
- Affordable SME-tier pricing: most platforms require enterprise contracts, leaving small manufacturers unserved
- AI-driven procedure quality validation: detecting ambiguity, missing steps, or unsafe sequences before publishing
- Automatic spatial anchor recovery in degraded tracking environments (reflective surfaces, low-texture walls)
- Cross-platform content portability: no open standard for exporting procedure content between competing platforms (vendor lock-in)
- GenAI natural language procedure authoring: describe a task in plain language; AI generates the structured procedure
- Proactive procedure suggestions: AI analyses sensor/MES data and recommends which procedure to run next
- Regulatory compliance mapping: automatically tag each step with the relevant ISO, GMP, or safety standard it satisfies
- Lightweight WebAR delivery without app installation — most platforms require native app installation

### AI-Augmentation Candidates
- Step-by-step instruction generation from unstructured expert video recordings (auto-caption, auto-segment)
- Visual inspection at each step (computer vision pass/fail) — partially addressed by PTC but widely applicable
- Adaptive procedures that reorder or skip steps based on worker skill level and real-time context
- Natural language query against procedure libraries ("How do I replace the O-ring on pump P-12?")
- Anomaly detection in completion data to surface hidden process failures before they become defects
- Auto-translation with domain-specific terminology preservation across step content

---

## Legal & IP Summary

All analysed solutions are fully proprietary commercial products. No open-source AR work instructions platforms were identified that offer comparable functionality. PTC Vuforia Engine has a free development-tier SDK but applies watermarking and usage restrictions in production. There are no identified patents on core AR work instruction delivery patterns (spatial anchoring, step-by-step overlay) that would preclude building an open-source alternative, though individual AI inspection algorithms and specific UI patterns may be patentable by their vendors. The use of glTF 2.0 (Khronos Group open standard), WebXR (W3C), and WebRTC (W3C/IETF) as foundational technology stack components for an open-source tool carries no IP risk. Integration with CAD formats (Siemens JT, PTC Creo, SolidWorks) requires either vendor-licensed SDKs or open-format intermediaries (STEP, glTF). No copyright concerns arise from building an independent platform that covers the feature categories documented here.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Web-based no-code procedure authoring: create, edit, and publish step-by-step instructions with text, images, and video
- Mobile AR playback: render AR overlays on iOS and Android via WebAR or a lightweight native app
- QR code / marker-based spatial anchoring for reliable equipment identification
- Offline procedure cache with sync-on-reconnect
- Completion tracking: per-step timestamps, operator ID, and pass/fail records stored for audit
- Role-based access control: author, reviewer/approver, and operator roles

**Should-have (v1.1)**
- 3D model (glTF) import and step-level 3D overlay in AR
- AI-powered SOP-to-procedure conversion: upload a PDF/Word SOP and generate a structured draft procedure
- Headset support: HoloLens 2 and/or Meta Quest via WebXR
- MES/ERP webhook integration: receive work order triggers to auto-assign procedures to operators
- Multi-language rendering: translate step content at runtime without duplicate procedure files
- Procedure versioning with approval workflow and rollback

**Nice-to-have (backlog)**
- AI visual inspection (computer vision step check) using device camera
- Remote expert video call with AR annotation capability
- Skills-based procedure assignment: match procedure complexity to operator competency profile
- CAD-system connectors (SolidWorks, Siemens NX, PTC Creo) for direct geometry import
- IoT sensor data overlay: display live equipment readings alongside instruction steps
- Air-gapped on-premises deployment option for defence/regulated environments
