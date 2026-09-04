# Botonera

Control panel software for OBS Studio, designed initially for a MacBook Air with Apple Silicon (M2 reference).

## Purpose

Provide a local software button panel for live production that sends reliable commands to OBS Studio without simulating mouse/keyboard clicks. The first version is intended for use during a livestream, with a UI optimized for fast, unambiguous operation.

## Initial controls

The first UI should expose these functions:

- **Q MUSICAL**
- **Q INTRO**
- **Q OUTRO**
- **TRANSICIÓN**
- **GRABAR**
- **DETENER**
- **HABLAR PRODUCTOR** — a configurable sequence that prepares and switches the production to the producer speaking.

These labels describe production functions, not necessarily the names of OBS scenes, sources, or devices.

## Critical implementation principles

1. **OBS is controlled through its supported WebSocket/API interface.** Do not use screen scraping, simulated mouse clicks, keyboard automation, or other fragile UI automation to operate OBS.
2. **Do not hardcode temporary OBS resource names.** Names such as a hypothetical “Micrófono 2” or “Cámara 2” are examples only and are NOT implementation requirements. The application must work with configurable resource identifiers and/or resources discovered from OBS.
3. **Separate production actions from OBS resource names.** `Q INTRO`, `Q OUTRO`, etc. are action names. Each action may eventually map to one or more OBS commands.
4. **Keep the application local.** The live control path must not depend on an internet connection.
5. **Keep the OBS integration isolated.** The UI must not contain OBS WebSocket calls directly; it should invoke an action/command layer.
6. **Design for future hardware inputs.** A future Stream Deck, Arduino, ESP32, Raspberry Pi, or other physical controller should be able to trigger the same action layer without duplicating OBS logic.
7. **Fail safely.** Connection loss, unavailable scenes/sources, or invalid configuration must produce a clear UI state/error rather than silently performing a different action.

## Expected high-level architecture

```text
UI / software buttons
        ↓
Production Action / Command layer
        ↓
OBS WebSocket client
        ↓
OBS Studio
```

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for the detailed boundaries and responsibilities.

## Producer speaking action

The producer-speaking control is intentionally specified as a **configurable action sequence**, not as fixed device names.

Conceptually it may need to:

1. make the producer camera visible/active;
2. unmute or otherwise enable the producer microphone;
3. switch to the appropriate producer scene/layout;
4. apply the configured transition, if required.

The exact OBS scene, camera source, microphone source, transition, and transition duration must be configurable or discovered from OBS. Do not infer permanent names from this document.

## Configuration and discovery

The application should have a configuration mechanism for mapping production actions to actual OBS resources. Where practical, it should query OBS for available scenes, sources, inputs, and transitions so configuration can reference resources that actually exist.

The implementation should avoid baking current production-specific names into source code.

## Initial scope

This repository is intentionally starting as documentation/specification only. Do not add implementation code until the implementation task is explicitly started.

The initial implementation should prioritize:

- reliable OBS connection;
- clear action-to-command separation;
- configurable OBS resource mapping;
- responsive button UI;
- visible connection/error state;
- safe behavior when an action cannot be completed.

Do not add unnecessary infrastructure, cloud services, remote backends, or internet dependencies unless a later requirement explicitly calls for them.

## Open decisions

The following are deliberately left open for implementation and should be chosen based on the simplest robust solution for macOS/Apple Silicon:

- programming language and UI framework;
- packaging/distribution method;
- exact visual design of the buttons;
- configuration file format and location;
- whether OBS discovery is fully automatic or assisted by a configuration UI;
- exact mappings from the seven production controls to OBS commands;
- exact transition type and duration for each action;
- whether buttons should support keyboard shortcuts in addition to mouse/touch input.

Codex must not invent production-specific OBS names or treat examples in this document as fixed values. If an implementation decision is genuinely required but not specified here, choose a minimal, maintainable default and document the decision.

## Implementation handoff for Codex

When implementation begins, Codex should first read `README.md` and `ARCHITECTURE.md`, inspect the repository state, and then implement the smallest complete vertical slice: establish the OBS connection, represent production actions independently of OBS resource names, and expose the initial controls through the UI. Keep the architecture extensible for future hardware inputs without implementing hardware support in the first version.
