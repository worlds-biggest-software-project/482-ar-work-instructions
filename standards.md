# Standards & API Reference

> Project: AR Work Instructions · Generated: 2026-05-07

---

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 12113:2022 — glTF 2.0 (Runtime 3D Asset Delivery Format)**
- URL: https://www.iso.org/standard/83990.html
- Khronos Group glTF 2.0 published as an ISO/IEC International Standard. The de-facto format for 3D asset exchange in AR/VR applications; used to deliver step-level 3D overlays in AR work instructions. Draco-compressed glTF is the practical production standard for industrial AR assets.

**ISO 10303 — STEP (Standard for the Exchange of Product Model Data)**
- URL: https://www.steptools.com/stds/step/
- The dominant neutral CAD data exchange format for manufacturing. AR work instruction platforms import STEP (.stp / .step) files to obtain assembly geometry from PLM systems (Siemens NX, PTC Creo, SolidWorks) without requiring format-specific SDKs.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Specifies requirements for a quality management system. Work instructions are explicitly referenced as documented information that supports conformance to ISO 9001; an AR work instructions platform that stores versioned, signed-off procedures supports ISO 9001 audit readiness.

**ISO 9241-11:2018 — Ergonomics of Human-System Interaction: Usability**
- URL: https://www.iso.org/standard/63500.html
- Defines usability concepts applicable to interactive systems. Relevant to the UX design of AR instruction overlays, where cognitive load, glanceability, and hands-free interaction must meet ergonomic standards for safe industrial use.

**ISO 9241-210:2019 — Human-Centred Design for Interactive Systems**
- URL: https://www.iso.org/standard/77520.html
- Provides requirements for human-centred design throughout the system lifecycle. AR work instruction authoring tools and player interfaces should comply with these principles to minimise worker fatigue and adoption resistance.

**IPC-7711/7721D:2024 — Rework, Modification and Repair of Electronic Assemblies**
- URL: https://webstore.ansi.org/standards/ipc/ipc77117721d2024
- Industry standard for documented rework procedures in electronics manufacturing. AR work instruction platforms targeting electronics and PCB assembly must be capable of rendering IPC-class procedures with sufficient visual precision.

---

### W3C & IETF Standards

**WebXR Device API (W3C Candidate Recommendation)**
- URL: https://www.w3.org/TR/webxr/
- The foundational W3C specification for AR and VR on the web. Enables `immersive-ar` sessions in browsers, allowing AR work instruction delivery without a native app install. Supported in Chrome 79+, Edge, Samsung Internet, and Safari on visionOS. The WebXR AR Module (Level 1) governs the `immersive-ar` session mode specifically.

**WebXR Augmented Reality Module — Level 1 (W3C)**
- URL: https://www.w3.org/TR/webxr-ar-module-1/
- Extends the WebXR Device API with AR-specific capabilities including the `immersive-ar` session mode, hit testing, and light estimation. Required for WebAR delivery of step overlays on mobile devices.

**WebRTC 1.0: Real-Time Communication Between Browsers (W3C/IETF)**
- URL: https://www.w3.org/TR/webrtc/
- Co-published W3C and IETF standard for peer-to-peer audio, video, and data communication in browsers. Powers the remote expert assistance feature in AR work instruction platforms — sub-300 ms latency on enterprise Wi-Fi. Supported in all major browsers with no plugin requirement.

**RFC 6749 — The OAuth 2.0 Authorization Framework (IETF)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorization framework for API access delegation. Required for integrating AR work instruction platforms with enterprise identity providers (Azure AD, Okta, Ping Identity). Combined with OpenID Connect (OIDC) for authentication.

**RFC 8288 — Web Linking (IETF)**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Specifies the `Link` header and relation types for hypermedia APIs. Relevant to REST API design for procedure resource navigation and pagination in AR work instruction backends.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.0 (OpenAPI Initiative)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- Vendor-neutral standard for machine-readable REST API description. All AR work instruction platforms expose REST APIs; OpenAPI 3.1 descriptions enable code generation of client SDKs, interactive documentation, and integration testing. Aligns with JSON Schema vocabularies for request/response validation.

**glTF 2.0 Specification (Khronos Group)**
- URL: https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html
- Royalty-free runtime 3D asset format. The primary container for 3D mesh, material, texture, and animation data used in AR instruction overlays. Supports Draco mesh compression extension for reduced payload size over constrained industrial networks. Backed by ISO/IEC 12113:2022.

**OPC UA (IEC 62541) — Unified Architecture (OPC Foundation)**
- URL: https://opcfoundation.org/about/opc-technologies/opc-ua/
- Cross-platform, open-source IEC standard for industrial data exchange from sensors to cloud. Enables AR work instruction platforms to display live machine telemetry (current speed, temperature, torque) alongside instruction steps. Supports client-server and publish-subscribe patterns over TCP/IP, MQTT, and HTTPS transports.

**GS1 Digital Link Standard**
- URL: https://www.gs1.org/standards/gs1-digital-link
- Standardised method for encoding GS1 product identifiers (GTIN, GLN, batch/serial numbers) into QR codes and 2D barcodes. AR work instruction platforms can scan GS1-compliant QR codes on equipment or assemblies to auto-identify the asset and launch the correct procedure — eliminating manual procedure selection.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + OpenID Connect 1.0**
- URL: https://oauth.net/2/ | https://openid.net/connect/
- Combined framework for API authorisation (OAuth 2.0) and federated identity authentication (OIDC). AR platforms must integrate with enterprise identity providers via OIDC for single sign-on and use OAuth 2.0 bearer tokens for API calls. Required for SOC 2 and ISO 27001 compliance in enterprise deployments.

**OWASP IoT Top 10**
- URL: https://owasp.org/www-project-internet-of-things/
- OWASP's documented top security risks for IoT/connected devices, including AR headsets and wearables. Key risks for AR work instruction deployments include insecure network services on headsets, lack of secure update mechanisms for device firmware, and insecure ecosystem interfaces (APIs). The OWASP Top 10:2025 (web application) is additionally relevant for the authoring web application.

**NIST SP 800-63B — Digital Identity Guidelines: Authentication**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- NIST guidance on authenticator assurance levels (AAL1–AAL3). Regulated manufacturing environments (pharma GMP, defence CMMC) may require AAL2 or AAL3 authenticators for procedure sign-off; the platform should support FIDO2/WebAuthn for phishing-resistant MFA.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- Open protocol for connecting AI models to external data sources and tools. Relevant to an AI-native AR work instructions platform: an MCP server exposing procedure content, completion records, and equipment data would allow LLM-based agents to query procedure libraries in natural language, generate new procedures from descriptions, or surface contextual guidance during execution.

---

## Similar Products — Developer Documentation & APIs

### PTC Vuforia Expert Capture

- **Description:** AR work instruction authoring and delivery platform with AI-powered visual inspection (Step Check) and 3D CAD integration. Part of the PTC ecosystem (Windchill PLM, ThingWorx IIoT).
- **API Documentation:** https://support.ptc.com/help/vuforia/editor/en/ (Help Centre; REST API access via direct PTC engagement)
- **SDKs/Libraries:** Vuforia Engine SDK (Unity/Native) — https://developer.vuforia.com/library/
- **Developer Guide:** https://developer.vuforia.com/library/
- **Standards:** REST/JSON; ThingWorx Integration
- **Authentication:** OAuth 2.0 / enterprise SSO via ThingWorx

### Scope AR WorkLink

- **Description:** Combined AR work instructions and real-time remote expert assistance platform. Supports HoloLens 2, Apple Vision Pro, Quest, iOS, and Android with native CAD import.
- **API Documentation:** https://www.scopear.com/developers/
- **SDKs/Libraries:** Not publicly documented; GraphQL API with enterprise access
- **Developer Guide:** https://help.scopear.com/hc/en-us/
- **Standards:** GraphQL API; REST webhooks; deep link integration
- **Authentication:** Company-level API tokens (non-expiring); OAuth 2.0 for enterprise SSO

### Augmentir

- **Description:** AI-native connected worker platform with GenAI (Augie) for procedure generation, skills management, and AI agents for industrial operations.
- **API Documentation:** https://www.augmentir.com/ (API details via direct engagement)
- **SDKs/Libraries:** Not publicly documented; oData and REST API connectors
- **Developer Guide:** Available to enterprise customers on request
- **Standards:** REST/JSON; oData for ERP integration; SAP, Oracle, UKG, Workday connectors
- **Authentication:** OAuth 2.0 / enterprise identity provider integration

### Microsoft Dynamics 365 Guides

- **Description:** HoloLens 2-native AR work instruction platform with gaze-controlled navigation, Power BI analytics, and real-time Teams collaboration.
- **API Documentation:** https://learn.microsoft.com/en-us/dynamics365/mixed-reality/guides/
- **SDKs/Libraries:** Dataverse Web API — https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/overview
- **Developer Guide:** https://learn.microsoft.com/en-us/dynamics365/mixed-reality/guides/developer-overview
- **Standards:** REST/OData 4 (Dataverse Web API); OpenAPI; Power BI REST API
- **Authentication:** Azure Active Directory / OIDC; OAuth 2.0

### Taqtile Manifest

- **Description:** AR work instruction and knowledge-capture platform for aerospace, defence, and industrial MRO. Supports air-gapped and on-premises deployment.
- **API Documentation:** https://taqtile.com/manifest/ (enterprise documentation available on request)
- **SDKs/Libraries:** Not publicly documented; IoT sensor data integration API
- **Developer Guide:** Available through enterprise onboarding
- **Standards:** REST/JSON; IoT sensor integration; defence security standards (ITAR/CMMC-aligned)
- **Authentication:** Enterprise-grade access control with encryption; supports SSO

### Frontline.io

- **Description:** XR training and remote support platform with a drag-and-drop Visual Workflow Editor and native CAD-to-digital-twin capability (Fast Track Pro).
- **API Documentation:** https://www.frontline.io/ (developer documentation not publicly listed)
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** Platform documentation available through customer portal
- **Standards:** REST/JSON; compatible with HoloLens 2, Quest 3, Vuzix Ultralight, RayNeo X3
- **Authentication:** Enterprise SSO; OAuth 2.0

### Proceedix (SymphonyAI)

- **Description:** Connected worker platform focused on offline-first digital procedures, conditional workflows, and compliance record generation. Backed by SymphonyAI industrial AI platform.
- **API Documentation:** https://www.proceedix.com/ (Open API for ERP/MES integration)
- **SDKs/Libraries:** REST API; available to enterprise customers
- **Developer Guide:** https://proceedix.com/platform/work-instructions
- **Standards:** REST/JSON Open API; ERP/MES integration
- **Authentication:** Enterprise SSO; OAuth 2.0

### Librestream Onsight

- **Description:** AI-powered AR platform for remote expert assistance with live AI translation in 27 languages, extreme low-bandwidth optimisation, and IoT sensor integration.
- **API Documentation:** https://librestream.com/platform/
- **SDKs/Libraries:** REST API; enterprise integration connectors
- **Developer Guide:** Available through enterprise onboarding
- **Standards:** REST/JSON; supports wearables (RealWear, Vuzix, ECOM Instruments)
- **Authentication:** Enterprise SSO; OAuth 2.0

### Microsoft Azure Spatial Anchors (Cloud Anchor Infrastructure)

- **Description:** Cross-platform (HoloLens, ARKit iOS, ARCore Android) cloud service for persisting spatial anchors across sessions and sharing them between devices and users.
- **API Documentation:** https://learn.microsoft.com/en-us/azure/spatial-anchors/
- **SDKs/Libraries:** Unity SDK, Android NDK SDK, C++/WinRT SDK — https://learn.microsoft.com/en-us/azure/spatial-anchors/
- **Developer Guide:** https://learn.microsoft.com/en-us/azure/spatial-anchors/overview
- **Standards:** REST/JSON management API; Azure Active Directory authentication
- **Authentication:** Azure AD / OAuth 2.0; API key for SDK sessions

### Apple ARKit (iOS/visionOS Spatial Anchors)

- **Description:** Apple's AR framework for iOS and visionOS providing surface detection, object recognition, persistent world anchors, and spatial sharing via `SharedCoordinateSpaceProvider` (WWDC25).
- **API Documentation:** https://developer.apple.com/documentation/arkit
- **SDKs/Libraries:** ARKit (Swift/Objective-C); RealityKit for rendering
- **Developer Guide:** https://developer.apple.com/documentation/arkit/content-anchors
- **Standards:** Apple proprietary; integrates with RealityKit, SceneKit; glTF import via Reality Converter
- **Authentication:** App-level; no separate API key required

---

## Notes

**Procedure content portability gap:** No open industry standard exists for AR work instruction content interchange. Each commercial platform uses a proprietary content model, creating significant vendor lock-in. A future open standard (analogous to SCORM for eLearning) would address this gap and enable open-source tooling to interoperate with commercial players.

**WebXR maturity:** WebXR AR Module is still a W3C Candidate Recommendation (not a full Recommendation), and browser support remains limited — Chrome and Edge on Android are the primary mobile targets. Safari/iOS support for `immersive-ar` is not available as of mid-2026 outside of visionOS, which constrains WebAR delivery on the dominant mobile platform.

**OPC UA for live sensor overlay:** OPC UA is the most widely supported industrial data standard for live telemetry from manufacturing equipment (PLCs, CNCs, robots). An open-source AR work instructions platform that integrates OPC UA natively would differentiate from competitors that require custom IoT connectors.

**CMMC/ITAR for defence:** AR work instruction platforms targeting defence customers must address CMMC Level 2/3 requirements and ITAR (International Traffic in Arms Regulations). Taqtile Manifest is currently the only commercially marketed AR work instruction tool with air-gapped/on-premises deployment explicitly designed for these requirements.
