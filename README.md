# Botonera

Control panel software for OBS Studio, designed initially for a MacBook Air with Apple Silicon (M2 reference).

## Mensaje para Gonza — implementación en Codex

Gonza: este repo es la especificación base para que mañana puedas abrirlo en Codex y hacer la implementación mientras Guille sigue con otras tareas.

Vos ya conocés la operación de producción; el objetivo de este documento es que Codex no tenga que reconstruir decisiones técnicas ni inventar nombres o comportamientos que no definimos.

**Importante:** la `Q` significa **Queue**. `Q MUSICAL`, `Q INTRO` y `Q OUTRO` son acciones de **poner en cola** el contenido correspondiente. No significa simplemente “botón Q” ni debe interpretarse como parte del nombre técnico de una escena de OBS.

También hay una aclaración clave sobre `HABLAR PRODUCTOR`: **cuando entra Gonza como productor, se lo agrega al video; NO se sacan Guille ni Marce del aire.** La acción debe sumar/habilitar la presencia del productor sobre la transmisión manteniendo a los conductores, salvo que una configuración futura indique explícitamente otra cosa.

### Contexto técnico del vivo

- Guille y Marce son los conductores.
- Gonza participa del programa y además opera la producción desde la Mac.
- El iPhone 16 es la cámara principal de los tres.
- JBL Quantum Stream Studio: audio de Guille + Marce.
- SM57 + interfaz: audio de Gonza.
- OBS Studio es el centro de video/audio, grabación y streaming.
- La botonera es una interfaz de control para que Gonza no tenga que navegar OBS durante la conversación.

La arquitectura general y el comportamiento de producción están documentados en `ARCHITECTURE.md`.

## Checklist del sábado

### Producción / contenidos

- [ ] Logo.
- [ ] Animación de intro.
- [ ] Animación de outro.

### OBS y dispositivos

- [ ] Configurar iPhone como cámara principal en OBS.
- [ ] Configurar JBL Quantum Stream Studio para Guille y Marce.
- [ ] Configurar cámara de la MacBook Air para Gonza.
- [ ] Configurar Shure SM57 + interfaz para el audio de Gonza.
- [ ] Verificar que cámara y micrófono del productor puedan prenderse/apagarse juntos mediante un botón de la botonera.
- [ ] El productor necesita monitor de video para ver la salida/imagen de OBS.
- [ ] El productor necesita escuchar el audio de OBS mediante auricular/monitorización adecuada, sin generar feedback al vivo.

### Macros de OBS / botonera

Cada macro de OBS debe quedar asociada a **un botón de la botonera**.

- [ ] Macro para prender/apagar cámara + micrófono del productor.
- [ ] Macro para ejecutar la intro.
- [ ] Macro para ejecutar el outro.
- [ ] Macro para el pie/cambio de sección.
- [ ] Verificar que cada botón dé feedback claro y que las acciones sean repetibles de forma segura.

### Integración con Codex — tarea de Gonza

Gonza debe:

1. Entrar a ChatGPT.
2. Abrir un **chat nuevo** específicamente para trabajar en la botonera.
3. Pasarle a ese chat el link de esta repo:
   urlhttps://github.com/JRRGUILLE-bit/Botonerahttps://github.com/JRRGUILLE-bit/Botonera
4. Pedirle a ChatGPT que lea la repo completa, especialmente `README.md` y `ARCHITECTURE.md`, y que lo ayude a usar **Codex** para implementar lo especificado ahí.
5. El objetivo de Codex es construir la **botonera web personalizada** para controlar OBS según esta arquitectura.
6. Antes de inventar cualquier comportamiento técnico, respetar las decisiones de producción documentadas en esta repo.
7. Probar la botonera contra el OBS local y dejar documentado cualquier decisión técnica que haya sido necesaria para completar la implementación.

**La implementación no debe convertirse en una botonera genérica ni en un simple mock visual. El objetivo es una botonera web personalizada, funcional y conectada a OBS.**

## Purpose

Provide a local software button panel for live production that sends reliable commands to OBS Studio without simulating mouse/keyboard clicks. The first version is intended for use during a livestream, with a UI optimized for fast, unambiguous operation.

## Initial controls

The first UI should expose these functions:

- **Q MUSICAL** — Queue Musical.
- **Q INTRO** — Queue Intro.
- **Q OUTRO** — Queue Outro.
- **TRANSICIÓN**
- **GRABAR**
- **DETENER**
- **HABLAR PRODUCTOR** — adds the producer to the video while keeping Guille and Marce on the video; the exact OBS sequence is configurable.

These labels describe production functions, not necessarily the names of OBS scenes, sources, or devices.

### Queue: significado y límite técnico

`Q` = **Queue**. Las tres acciones de queue representan la intención de producción de poner en cola Musical, Intro u Outro.

El documento **no asume que OBS tenga un comando nativo llamado Queue**. El mecanismo concreto para materializar esa cola en OBS debe definirse mediante configuración y/o la lógica de producción que se implemente. Codex no debe inventar una interpretación técnica de Queue solamente a partir del nombre del botón.

## Critical implementation principles

1. **OBS is controlled through its supported WebSocket/API interface.** Do not use screen scraping, simulated mouse clicks, keyboard automation, or other fragile UI automation to operate OBS.
2. **Do not hardcode temporary OBS resource names.** Names such as a hypothetical “Micrófono 2” or “Cámara 2” are examples only and are NOT implementation requirements. The application must work with configurable resource identifiers and/or resources discovered from OBS.
3. **Separate production actions from OBS resource names.** `Q INTRO`, `Q OUTRO`, etc. are action names. Each action may eventually map to one or more OBS commands.
4. **Keep the application local.** The live control path must not depend on an internet connection.
5. **Keep the OBS integration isolated.** The UI must not contain OBS WebSocket calls directly; it should invoke an action/command layer.
6. **Design for future hardware inputs.** A future Stream Deck, Arduino, ESP32, Raspberry Pi, or other physical controller should be able to trigger the same action layer without duplicating OBS logic.
7. **Fail safely.** Connection loss, unavailable scenes/sources, or invalid configuration must produce a clear UI state/error rather than silently performing a different action.
8. **Do not remove the hosts when adding the producer.** `HABLAR PRODUCTOR` means adding/enabling Gonza in the video while Guille and Marce remain present.

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

`HABLAR PRODUCTOR` is a **configurable production action**. The important production behavior is:

> Add Gonza to the video; do not remove Guille or Marce.

Conceptually it may need to:

1. make the producer camera visible/active in the current video composition;
2. unmute or otherwise enable the producer microphone;
3. preserve the host video/audio presence;
4. apply the configured transition or layout change, if required.

The exact OBS scene, camera source, microphone source, scene-item visibility, transition, and transition duration must be configurable or discovered from OBS. Do not infer permanent names from this document.

If the implementation needs a dedicated scene/layout to achieve this, that scene must contain the hosts as well as the producer. A producer scene that replaces the hosts is **not** the intended behavior.

## Configuration and discovery

The application should have a configuration mechanism for mapping production actions to actual OBS resources. Where practical, it should query OBS for available scenes, sources, inputs, scene items, and transitions so configuration can reference resources that actually exist.

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
- exact technical implementation of Queue actions;
- exact transition type and duration for each action;
- exact OBS resources used to add Gonza while preserving the hosts;
- whether buttons should support keyboard shortcuts in addition to mouse/touch input.

Codex must not invent production-specific OBS names or treat examples in this document as fixed values. If an implementation decision is genuinely required but not specified here, choose a minimal, maintainable default and document the decision. If the decision affects the intended production behavior, flag it clearly rather than silently changing the behavior.

## Implementation handoff for Codex

When implementation begins, Codex should:

1. Read `README.md` and `ARCHITECTURE.md` completely.
2. Inspect the repository state before changing anything.
3. Preserve the production semantics documented here, especially **Q = Queue** and **HABLAR PRODUCTOR = add Gonza without removing Guille or Marce**.
4. Implement the smallest complete vertical slice: establish the OBS connection, represent production actions independently of OBS resource names, and expose the initial controls through the UI.
5. Keep OBS integration isolated from the UI.
6. Make production-specific OBS resources configurable/discoverable instead of hardcoding guesses.
7. Test the actions against a real local OBS instance where possible.
8. Document any implementation decision that was not explicitly specified.
9. Keep the architecture extensible for future hardware inputs without implementing hardware support in the first version.

**Do not stop at a mock UI if the local OBS connection and core actions can be implemented and tested. The objective is a usable V1, not only a visual prototype.**
