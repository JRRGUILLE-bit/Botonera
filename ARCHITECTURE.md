# Architecture

## 0. Mensaje de producción para Gonza

Gonza es **participante + productor**. La botonera se diseña para que pueda operar OBS sin dejar de participar de la conversación.

Dos reglas de producción son fundamentales y no deben perderse durante la implementación:

1. **`Q` significa Queue.** `Q MUSICAL`, `Q INTRO` y `Q OUTRO` representan acciones de poner en cola el contenido correspondiente. Queue es una intención de producción; no debe asumirse que existe un comando nativo de OBS llamado `Queue`.
2. **`HABLAR PRODUCTOR` agrega a Gonza al video y NO saca a Guille ni a Marce.** Si se utiliza una escena/layout alternativo, esa composición debe conservar a los dos conductores y sumar al productor. No implementar una escena de “solo productor” como interpretación de este botón.

Los nombres concretos de cámaras, micrófonos, escenas y fuentes todavía no son requisitos fijos. Codex debe hacerlos configurables o descubrirlos desde OBS.

## 1. Architectural goal

The application is a local control surface for OBS Studio. Its architecture must keep the production intent of a button separate from the technical details required to execute that intent in OBS.

The key design rule is:

> A production action must not know or depend on a hardcoded OBS scene/source/device name.

This makes the application configurable for different OBS setups and allows future physical controllers to reuse the same control logic.

## 2. Production context

The V1 livestream setup is:

```text
GUILLE — host
MARCE  — host
GONZA  — participant + producer

Video: iPhone 16
Host audio: JBL Quantum Stream Studio
Producer audio: SM57 -> audio interface -> Mac
Control: software button panel -> OBS WebSocket -> OBS
```

The iPhone is the main camera for the three people. The producer button is therefore primarily a **composition/state change** that adds Gonza to the existing program rather than replacing the hosts.

The broader technical production document supplied for this project also defines the intended OBS baseline: 1920x1080 canvas/output, 30 fps, 48 kHz audio, local MKV recording, separate audio tracks where possible, and local operation without relying on internet services for control.

## 3. Components

### 3.1 UI layer

Responsible only for presentation and user interaction.

Responsibilities:

- Render the production controls.
- Show active/inactive state where useful.
- Show OBS connection state.
- Provide clear feedback for success/failure.
- Trigger named production actions.

The UI must not contain OBS WebSocket protocol calls or OBS-specific resource lookup logic.

Initial controls:

- `Q MUSICAL` — Queue Musical
- `Q INTRO` — Queue Intro
- `Q OUTRO` — Queue Outro
- `TRANSICIÓN`
- `GRABAR`
- `DETENER`
- `HABLAR PRODUCTOR`

### 3.2 Production action layer

Represents what the operator wants to accomplish rather than how OBS accomplishes it.

Conceptual internal actions should use unambiguous names such as:

- `QUEUE_MUSICAL`
- `QUEUE_INTRO`
- `QUEUE_OUTRO`
- `TRANSITION`
- `START_RECORDING`
- `STOP_RECORDING`
- `PRODUCER_SPEAKING`

**Queue semantics:** `QUEUE_*` means “put this production content in the queue.” The architecture must not invent a specific OBS command for Queue. The actual implementation may involve scene/source/media state changes, a local queue/state model, or another configured sequence depending on how the production workflow is defined. That mapping belongs in configuration/action logic, not in the button label.

An action may translate into one OBS command or a sequence of commands.

### 3.3 Producer speaking action

`PRODUCER_SPEAKING` has a specific production requirement:

> Add Gonza to the video while keeping Guille and Marce in the video.

A possible implementation sequence is:

```text
make producer camera visible/active
        ↓
unmute/enable producer microphone
        ↓
preserve host camera/video presence
        ↓
apply configured composition/layout change
        ↓
apply configured transition if required
```

This is behavioral guidance, not a demand for a particular OBS scene structure. The implementation may use scene switching, scene-item visibility, source activation, or another OBS mechanism. What matters is the resulting behavior: **Gonza is added; the hosts remain.**

If a dedicated scene is used, it must contain the hosts plus the producer. A scene that replaces the hosts with Gonza is incorrect for this action.

The exact OBS scene, camera source, microphone source, scene-item IDs, transition, and transition duration must be configurable or discovered from OBS.

### 3.4 Configuration layer

Stores the mapping between production actions and the actual OBS resources/commands required by the current setup.

It should be possible to change the OBS setup without recompiling the application.

Configuration may eventually describe things such as:

```text
QUEUE_INTRO -> configured queue behavior
QUEUE_OUTRO -> configured queue behavior
START_RECORDING -> OBS StartRecord
PRODUCER_SPEAKING -> configured producer camera + mic + host-preserving layout + transition
```

The exact schema is intentionally open for implementation.

### 3.5 OBS discovery layer

Optional but strongly preferred.

The application should be able to query OBS through WebSocket for available resources, such as:

- scenes;
- scene items;
- input/source names;
- input settings where relevant;
- transitions;
- OBS state needed by the actions.

Discovery should help configuration reference real resources instead of relying on guessed or stale names.

For `HABLAR PRODUCTOR`, discovery/configuration should make it possible to identify the producer camera and microphone and verify that the selected video composition also contains the hosts.

### 3.6 OBS WebSocket client

This is the only layer that should know the OBS WebSocket protocol and request/response details.

Responsibilities:

- establish and maintain the local OBS connection;
- authenticate if OBS requires it;
- send OBS requests;
- receive OBS events/state changes;
- expose a clean internal interface to the action layer;
- report connection and command errors.

The rest of the application should not need to know WebSocket message formats.

### 3.7 Future hardware input layer

Not part of the first implementation.

The architecture should nevertheless allow a physical controller to send the same production actions as the software UI:

```text
Software button ─┐
                 ├─> Production Action Layer ─> OBS WebSocket ─> OBS
Future hardware ─┘
```

Possible future inputs include Stream Deck, Arduino, ESP32, Raspberry Pi, MIDI, or another controller. No hardware-specific code should be required in the OBS integration layer.

## 4. Runtime flow

A normal button press should follow this sequence:

```text
Operator presses UI button
        ↓
UI emits production action
        ↓
Action layer resolves configured action
        ↓
Action layer invokes OBS client
        ↓
OBS WebSocket sends request(s)
        ↓
OBS executes request(s)
        ↓
Result/event returns to application
        ↓
UI reflects success, state, or error
```

For multi-step actions, the action layer owns the sequence and should handle failures explicitly.

For Queue actions, the action layer must preserve the production meaning of “put this content in queue” rather than treating the Q as a generic scene-switch shortcut.

## 5. Resource naming rule

Names such as `Micrófono 2`, `Cámara 2`, `Escena Productor`, `INTRO`, etc. must be treated as examples unless they are explicitly configured by the user.

They must never be embedded as assumptions in the architecture or implementation.

Production labels and OBS resource identifiers are different concepts:

```text
Production intent:  HABLAR PRODUCTOR
                       ↓
Configuration:       producer_scene/layout = <host-preserving composition>
                     producer_camera       = <actual OBS source>
                     producer_mic          = <actual OBS input>
                     transition            = <actual OBS transition>
                       ↓
OBS commands
```

The `<host-preserving composition>` requirement is functional: whatever OBS resources are selected, Guille and Marce must remain in the video when Gonza is added.

## 6. Reliability requirements

Because the application is intended for live production:

- Do not use simulated mouse clicks or keyboard automation against OBS.
- Do not depend on an internet service for the local control path.
- Detect and display OBS connection loss.
- Do not silently substitute an unknown scene/source when configuration is invalid.
- Make multi-step actions explicit and failure-aware.
- Avoid blocking the UI while waiting for OBS responses.
- Keep logs useful for diagnosing connection and command failures.
- Prefer deterministic behavior over clever automation.
- Protect destructive/end actions from accidental presses where appropriate.
- Give the operator immediate visible feedback after an action.
- Avoid double-triggering a multi-step action from accidental rapid presses unless the action is explicitly designed to be repeatable.

## 7. macOS / Apple Silicon target

The reference machine is a MacBook Air with an Apple M2 chip.

The first implementation must run natively and reliably on modern macOS/Apple Silicon. Framework and dependency choices should favor mature Apple Silicon support and a simple local installation/launch process.

Do not introduce cross-platform complexity unless it provides a concrete benefit to this project.

## 8. Separation of concerns

The following dependencies are desirable:

```text
UI
 ↓
Action interface
 ↓
OBS service interface
 ↓
OBS WebSocket implementation
```

And configuration should be consumed by the action layer rather than embedded in UI components.

Avoid:

```text
UI button → hardcoded OBS scene name → raw WebSocket request
```

That pattern makes the application difficult to maintain and prevents future hardware inputs from sharing the same behavior.

## 9. Initial implementation boundary

The first implementation should not attempt to solve every future requirement.

Minimum useful vertical slice:

1. Start the application locally on macOS/Apple Silicon.
2. Connect to a running OBS Studio instance through its WebSocket interface.
3. Represent production actions independently from OBS resource names.
4. Implement configuration for the initial actions.
5. Render the seven initial controls.
6. Execute at least the core OBS actions reliably.
7. Show connection and command errors clearly.
8. Implement `HABLAR PRODUCTOR` so the configured producer camera/mic are added/enabled without removing the hosts.
9. Keep Queue actions represented as Queue production intents, with their concrete OBS mapping configurable.

Do not implement physical hardware support, remote/cloud control, or unnecessary backend infrastructure in the first version.

## 10. Decisions intentionally left open

Codex may choose the language, UI framework, project structure, configuration format, and packaging approach, provided the choices satisfy this architecture.

When a choice is not specified:

1. prefer the simplest robust solution;
2. prefer native/reliable macOS and Apple Silicon support;
3. minimize dependencies;
4. keep OBS integration isolated;
5. document significant decisions.

If a production-specific detail is missing, do not invent a permanent OBS resource name. Make it configurable or clearly mark it as an implementation decision requiring configuration.

Do not reinterpret `Q` or change the producer behavior to simplify implementation.
