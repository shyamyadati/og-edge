# Architecture

This document explains how a running OpenGadget system works. For what the project is and how to use it, start with the [README](../README.md).

In this document, a **sensor** is a camera, 3D line scanner, or any other visual inspection system that has an API or software library associated with it.

## How an inspection runs

Here's what happens when a part passes the sensor:

```text
Trigger ─► Sensor ─► Sensor service ─► Pipeline ─► Automation service ─► PLC
                            │               │               │
                            ▼               ▼               ▼
          Platform: messages, in-memory images, results, telemetry, retained sensor data
```

1. **Trigger:** The part trips a photo eye wired directly to the sensor, so capture timing stays in hardware.
2. **Capture:** The sensor service receives the image, tags it with a unique `event_id`, stores it in memory, and publishes a message saying where to find it.
3. **Inspect:** The pipeline receives that message, reads the image, runs your models and logic, and publishes a result.
4. **Act:** The automation service turns the result into whatever your machine expects and writes it to the PLC.
5. **Record:** Results, timing, and selected images are saved, all linked by `event_id`.

Every service reacts to messages and communicates only through the platform. Adding another sensor or pipeline therefore means adding services and connecting them to the right topics, without changing the ones already running:

```text
Sensor A ──┐           ┌─► Pipeline 1 ──┐
           ├─► Topics ─┤                ├─► Automation X ─► PLC/Factory Automation
Sensor B ──┤           ├─► Pipeline 2 ──┘
           │           │
Sensor C ──┘           └─► Pipeline 3 ────► Automation Y ─► PLC/Factory Automation
```

Each service runs separately, so a crash or slowdown in one inspection doesn't stall the others, and you can update one pipeline while the rest keep running.

## What you get and what gets generated

| Part                | Who provides it                     | What it includes                                                   |
| ------------------- | ----------------------------------- | ------------------------------------------------------------------ |
| Platform            | OpenGadget                          | The services on every device, described in [Platform](platform.md) |
| Sensor service      | OpenGadget, configured by the agent | Capture from any GigE Vision or GenICam sensor                     |
| Design constraints  | OpenGadget                          | Rules and tests that define what a correct custom service does     |
| Agent tooling       | OpenGadget                          | Skills, hooks, agent instructions, and a command-line tool         |
| Pipelines           | Generated for you                   | Your models and inspection logic                                   |
| Automation services | Generated for you                   | The connection to your PLCs and automation equipment               |
| HMI                 | Generated for you                   | The operator screen for your inspection                            |
| PLC code            | Generated for your specialist       | Structured text or ladder logic that receives the results          |

## Sensors and triggering

OpenGadget supports GigE Vision and GenICam sensors through one generic service, so adding one means changing its settings rather than writing code. Software triggers are available for setup and testing. An emulation mode plays test images through the sensor service, so you can test without a sensor.

## `event_id` coordination

Sensors that share a trigger form a group, and each group works in numbered rounds so their services hold an ID before the trigger fires:

1. Each sensor service clears its sensor's buffers and joins the group.
2. The event ID service publishes a round number and a new `event_id`.
3. Each sensor service takes the ID and confirms it's ready. Once all have confirmed, the service generates the next ID.
4. The trigger fires, and each sensor service tags its image with the ID it holds.
5. Each sensor service moves to the next round.

If a sensor service misses a round, the service raises an alert, and the event follows the inspection's failure policy. Sensor frame counters provide a second check for dropped images. To recover, the group repeats the first three steps. Coordination covers the sensors on a single device.

## Messages

Messages are JSON, so any service can process them. Each has a schema version, message type, source service, microsecond timestamp, and `event_id`. Image messages also have the image's key and dimensions, and the image itself is never in the message. Other fields are free-form.

Messages that affect an inspection use Valkey Streams, which let a restarted service catch up and allow replay during debugging. Other messages go through the database API to Postgres, so a slow database can't delay an inspection.

## Image storage

Images in process live in a separate Valkey instance from messages. Messages must never be evicted, while images can expire, and separating them keeps large image writes from delaying messages. The data manager saves images to disk according to your policies, such as keeping every defect for 90 days but only a sample of good parts.

## Time

Timestamps are integer microseconds. Wall-clock time records when something happened, and a monotonic clock measures durations, because the wall clock can jump when corrected. Image messages also carry the sensor's hardware timestamp when one is available.

## Resources on a device

Parallel inspections share the GPU, memory, image storage, and network bandwidth. Each inspection declares what it needs, and the agent checks that everything fits before deploying. Each physical output belongs to one automation service, and if several pipelines feed it, a configuration rule says how their results combine.

## Languages

OpenGadget uses the right technology for each component rather than one language throughout:

| Technology | Where it fits                                                                           |
| ---------- | --------------------------------------------------------------------------------------- |
| Rust       | Components close to hardware, or where memory safety and predictable timing matter most |
| Go         | Network services and command-line tooling                                               |
| TypeScript | The status page, HMI, on-demand views, and MCP tooling                                  |
| Python     | Models and pipelines, because most model tooling is in Python                           |

Generated automation services use whichever language suits the machine's protocol and libraries. Services work together regardless of language because messages are JSON and the same conformance tests apply to all of them.

## Related documents

- [Platform](platform.md) lists the services on every device.
- [Custom services](custom-services.md) explains how pipelines and other services are built.
- [Automation](automation.md) explains the connection to your machines.
- [Operations](operations.md) explains failure handling, monitoring, and updates.
- The [risk register](risks.md) lists what's still unproven or undecided.