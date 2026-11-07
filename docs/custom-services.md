# Custom Services

This document explains how the custom parts of an OpenGadget system are built, including the code your agent generates and the rules it follows. For how these services fit into the running system, see the [architecture](architecture.md).

You won't usually write these yourself, but it helps to know what your agent is producing. Every service is built from the same layers, so they all look alike and any agent can read and maintain them:

```text
MessageBase      Message fields, JSON conversion, and validation
InterfaceBase    A client connection that sends and receives one message type
ServiceBase      The five interfaces below, configuration, and start/stop
  ├─ Sensor, Pipeline, Automation, or another service type
  │    └─ Your custom logic
```

The custom layer is small:

| Service type | Your code implements                                                                                 |
| ------------ | ---------------------------------------------------------------------------------------------------- |
| Pipeline     | `load`, `warm_up`, and `predict`, plus a manifest listing its models                                 |
| Automation   | `connect`, `disconnect`, `is_connected`, `map_result`, `write_result`, and any optional background tasks |
| HMI | A screen for the inspection, using the contract for its data and operator actions |

Each service also has a definition file that lists its settings, their default values, and any handler to run when a value changes. This lets the MCP server discover what can be changed on any service.

The layers below your code provide five interfaces:

| Interface            | What it does                                                                           |
| -------------------- | -------------------------------------------------------------------------------------- |
| Data messaging       | Sends and receives inspection messages with low latency                                |
| Data storage         | Reads and writes images and metadata                                                   |
| Telemetry and status | Publishes health, performance, and timing                                              |
| Alerts               | Publishes critical alerts, separately from routine telemetry                           |
| Configuration        | Applies setting changes on its own thread, leaving the main thread free for processing |

They also handle timestamps, `event_id`, and message formatting, so your custom code doesn't have to.

## Design constraints and tests

The `constraints/` folder tells agents what a correct service does: its contract, messages, naming, telemetry, and failure behavior, along with recommended model designs and their licenses. The `conformance/` folder contains tests that check those rules. `examples/` has working services for reference.

## Models

Labeling and training use existing open source tools and run off the device, on a local GPU or in the cloud. A pipeline asks for a model role, and the model manager determines which model fills that role. Check model licenses before use, since OpenGadget is MIT licensed. For example, Ultralytics YOLO is AGPL-3.0, while YOLOX is permissively licensed.
