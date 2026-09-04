# Architecture

## 1. Architectural goal

The application is a local control surface for OBS Studio. Its architecture must keep the production intent of a button separate from the technical details required to execute that intent in OBS.

The key design rule is:

> A production action must not know or depend on a hardcoded OBS scene/source/device name.

This makes the application configurable for different OBS setups and allows future physical controllers to reuse the same control logic.

## 2. Components

### 2.1 UI layer

Responsible only for presentation and user interaction.

Responsibilities:

- Render the production controls.
- Show active/inactive state where useful.
- Show OBS connection state.
- Provide clear feedback for success/failure.
- Trigger named production actions.

The UI must not contain OBS WebSocket protocol calls or OBS-specific resource lookup logic.

Initial controls:

- `Q MUSICAL`
- `Q INTRO`
- `Q OUTRO`
- `TRANSICIÓN`
- `GRABAR`
- `DETENER`
- `HABLAR PRODUCTOR`

### 2.2 Production action layer

Represents what the operator wants to accomplish rather than how OBS accomplishes it.

Examples of conceptual actions:

- `Q_MUSICAL`
- `Q_INTRO`
- `Q_OUTRO`
- `TRANSITION`
- `START_RECORDING`
- `STOP_RECORDING`
- `PRODUCER_SPEAKING`

An action may translate into one OBS command or a sequence of commands.

For example, `PRODUCER_SPEAKING` may resolve to a sequence such as:

```text
activate/show producer camera source
unmute/enable producer microphone source
switch to producer scene/layout
apply configured transition
```

This is a behavioral example, not a fixed implementation. The actual resources must come from configuration and/or OBS discovery.

### 2.3 Configuration layer

Stores the mapping between production actions and the actual OBS resources/commands required by the current setup.

It should be possible to change the OBS setup without recompiling the application.

Configuration may eventually describe things such as:

```text
Q_INTRO -> switch to configured intro scene
Q_OUTRO -> switch to configured outro scene
START_RECORDING -> OBS StartRecord
PRODUCER_SPEAKING -> configured scene + configured mic + configured camera + transition
```

The exact schema is intentionally open for implementation.

### 2.4 OBS discovery layer

Optional but strongly preferred.

The application should be able to query OBS through WebSocket for available resources, such as:

- scenes;
- scene items;
- input/source names;
- input settings where relevant;
- transitions;
- OBS state needed by the actions.

Discovery should help configuration reference real resources instead of relying on guessed or stale names.

### 2.5 OBS WebSocket client

This is the only layer that should know the OBS WebSocket protocol and request/response details.

Responsibilities:

- establish and maintain the local OBS connection;
- authenticate if OBS requires it;
- send OBS requests;
- receive OBS events/state changes;
- expose a clean internal interface to the action layer;
- report connection and command errors.

The rest of the application should not need to know WebSocket message formats.

### 2.6 Future hardware input layer

Not part of the first implementation.

The architecture should nevertheless allow a physical controller to send the same production actions as the software UI:

```text
Software button ─┐
                 ├─> Production Action Layer ─> OBS WebSocket ─> OBS
Future hardware ─┘
```

Possible future inputs include Stream Deck, Arduino, ESP32, Raspberry Pi, MIDI, or another controller. No hardware-specific code should be required in the OBS integration layer.

## 3. Runtime flow

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

## 4. Resource naming rule

Names such as `Micrófono 2`, `Cámara 2`, `Escena Productor`, `INTRO`, etc. must be treated as examples unless they are explicitly configured by the user.

They must never be embedded as assumptions in the architecture or implementation.

Production labels and OBS resource identifiers are different concepts:

```text
Production intent:  HABLAR PRODUCTOR
                       ↓
Configuration:       producer_scene = <actual OBS scene>
                     producer_camera = <actual OBS source>
                     producer_mic = <actual OBS input>
                     transition = <actual OBS transition>
                       ↓
OBS commands
```

## 5. Reliability requirements

Because the application is intended for live production:

- Do not use simulated mouse clicks or keyboard automation against OBS.
- Do not depend on an internet service for the local control path.
- Detect and display OBS connection loss.
- Do not silently substitute an unknown scene/source when configuration is invalid.
- Make multi-step actions explicit and failure-aware.
- Avoid blocking the UI while waiting for OBS responses.
- Keep logs useful for diagnosing connection and command failures.
- Prefer deterministic behavior over clever automation.

## 6. macOS / Apple Silicon target

The reference machine is a MacBook Air with an Apple M2 chip.

The first implementation must run natively and reliably on modern macOS/Apple Silicon. Framework and dependency choices should favor mature Apple Silicon support and a simple local installation/launch process.

Do not introduce cross-platform complexity unless it provides a concrete benefit to this project.

## 7. Separation of concerns

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

## 8. Initial implementation boundary

The first implementation should not attempt to solve every future requirement.

Minimum useful vertical slice:

1. Start the application locally on macOS/Apple Silicon.
2. Connect to a running OBS Studio instance through its WebSocket interface.
3. Represent production actions independently from OBS resource names.
4. Implement configuration for the initial actions.
5. Render the seven initial controls.
6. Execute at least the core OBS actions reliably.
7. Show connection and command errors clearly.

Do not implement physical hardware support, remote/cloud control, or unnecessary backend infrastructure in the first version.

## 9. Decisions intentionally left open

Codex may choose the language, UI framework, project structure, configuration format, and packaging approach, provided the choices satisfy this architecture.

When a choice is not specified:

1. prefer the simplest robust solution;
2. prefer native/reliable macOS and Apple Silicon support;
3. minimize dependencies;
4. keep OBS integration isolated;
5. document significant decisions.

If a production-specific detail is missing, do not invent a permanent OBS resource name. Make it configurable or clearly mark it as an implementation decision requiring configuration.
