# OpenGadget Architecture

OpenGadget helps you build a visual inspection system for a production line by working with an AI coding agent. You describe your cameras, your machines, and what you want to inspect. The agent configures the prepackaged parts, writes the custom code, and tests it with you. The result runs on an NVIDIA Jetson or an x86 Linux computer with an NVIDIA GPU, and keeps running with no AI involved.

This document explains how the system works and how it's organized. It's for anyone evaluating OpenGadget, building an inspection system with it, or contributing to it.

> **Project status:** OpenGadget is in early design, and nothing is built yet. This document describes what we're building toward, and details may change. See the [risk register](risks.md) for open decisions and risks.

## What makes it different

- **You work with an agent instead of a configuration screen.** The agent does the setup and coding in your own repository, and you review the results at each step.
- **The agent only writes what's specific to you.** Camera capture, messaging, storage, and monitoring come ready to use. The agent writes your inspection logic and the connection to your machines.
- **Your line never depends on AI.** Only ordinary, deterministic software runs on the device. If the agent, the network, or the model provider goes away, inspection keeps running.
- **Generated code has to pass tests.** A service can't be deployed until it passes tests covering its messages, health reporting, and failure behavior.
- **You can run many inspections on one device.** A single device runs multiple cameras, pipelines, and machine connections in parallel.

## Building an inspection system

1. **Create your repository.** Use the OpenGadget GitHub template. The new repository is yours, and it can be private.
2. **Start your agent.** Open an AI coding agent such as Claude or Codex in the repository and describe your setup. For example: "Let's build an inspection system that connects to a GigE Vision camera."
3. **Build and install.** Build a Debian package on your computer, copy it to your inspection device, and start the system.
4. **Build up in stages.** Add one piece at a time with your agent, checking each one before you move on.

| Stage          | What you do with the agent                           | How you know it worked                                   |
| -------------- | ---------------------------------------------------- | -------------------------------------------------------- |
| 1. Base system | Install and start it on the device                   | The device's status page shows healthy services          |
| 2. Sensor      | Discover and configure your camera                   | Triggered images appear                                  |
| 3. Data        | Collect sample images                                | You have a usable set of good and bad parts              |
| 4. Pipeline    | Generate and test the inspection                     | Results are correct on known samples and live images     |
| 5. Automation  | Generate the machine connection and PLC code         | Results arrive correctly at your PLC                     |
| 6. HMI | Generate the operator screen for the inspection | Operators can see results and faults at the line |
| 7. Expand      | Add more cameras, pipelines, and machine connections | Everything passes together, including failure tests      |
| 8. Operate     | Monitor, maintain, and retrain                       | Problems get found and fixed through changes you approve |

Your repository holds your configuration, code, PLC integration details, and history, so you or a future agent can see how the system got to its current state and roll back a change.

## How an inspection runs

Here's what happens when a part passes the camera:

```text
Trigger ─► Camera ─► Sensor service ─► Pipeline ─► Automation service ─► PLC
                            │               │               │
                            ▼               ▼               ▼
          Base system: messages, in-memory images, results, telemetry, retained images
```

1. **Trigger:** The part trips a photo eye wired directly to the camera, so capture timing stays in hardware.
2. **Capture:** The sensor service receives the image, tags it with a unique `event_id`, stores it in memory, and publishes a message saying where to find it.
3. **Inspect:** The pipeline receives that message, reads the image, runs your models and logic, and publishes a result.
4. **Act:** The automation service turns the result into whatever your machine expects and writes it to the PLC.
5. **Record:** Results, timing, and selected images are saved, all linked by `event_id`.

Every service reacts to messages and communicates only through the base system. Adding another camera or pipeline therefore means adding services and connecting them to the right topics, without changing the ones already running:

```text
Camera A ──┐           ┌─► Pipeline 1 ──┐
           ├─► Topics ─┤                ├─► Automation X ─► PLC
Camera B ──┤           ├─► Pipeline 2 ──┘
           │           │
Camera C ──┘           └─► Pipeline 3 ────► Automation Y ─► PLC
```

Each service runs separately, so a crash or slowdown in one inspection doesn't stall the others, and you can update one pipeline while the rest keep running.

## What you get and what gets generated

| Part                | Who provides it                     | What it includes                                               |
| ------------------- | ----------------------------------- | -------------------------------------------------------------- |
| Base system         | OpenGadget                          | The services on every device, described below                  |
| Sensor service      | OpenGadget, configured by the agent | Capture from any GigE Vision or GenICam camera                 |
| Design constraints  | OpenGadget                          | Rules and tests that define what a correct custom service does |
| Agent tooling       | OpenGadget                          | Skills, hooks, agent instructions, and a command-line tool     |
| Pipelines           | Generated for you                   | Your models and inspection logic                               |
| Automation services | Generated for you                   | The connection to your PLCs and automation equipment           |
| HMI | Generated for you | The operator screen for your inspection |
| PLC code            | Generated for your specialist       | Structured text or ladder logic that receives the results      |

### Services on the device

| Service          | What it does for you                                                                |
| ---------------- | ----------------------------------------------------------------------------------- |
| Valkey           | Carries messages and holds images in memory while they're being processed           |
| Postgres         | Stores results, telemetry, configuration, and audit logs                            |
| Database API     | Gives services a common way to use Postgres, so generated code doesn't need SQL     |
| Event ID service | Keeps `event_id` in step across cameras that share a trigger                        |
| Data manager     | Archives images to disk and prunes old data according to your retention rules       |
| Model manager    | Tracks which model is in use for each role and prepares models to run on the device |
| System monitor   | Reports on storage, memory, GPU, and other device health                            |
| Logging service  | Records telemetry and status so you can trace what happened                         |
| MCP server       | Lets AI agents inspect and manage the device, within the permissions you set        |
| Status page      | Shows device and inspection status to people on site, with no agent or internet     |

Generated services run in containers, so a faulty one can't take down the base system. A prebuilt image for each supported platform supplies the drivers and inference runtime for pipelines, so a pipeline doesn't have to handle hardware differences itself.

## Writing custom services

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

### Design constraints and tests

The `constraints/` folder tells agents what a correct service does: its contract, messages, naming, telemetry, and failure behavior, along with recommended model designs and their licenses. The `conformance/` folder contains tests that check those rules. `examples/` has working services for reference.

## Connecting to your machines

An automation service connects the inspection system to your PLCs and automation equipment. These are generated because every machine integration is different, and you can regenerate one quickly when a machine changes.

**Your PLC stays in charge of tracking product.** Whether your line uses fixed delays, specific registers, or queues inside the PLC, OpenGadget writes results where the PLC expects them. It can also write `event_id`, which your PLC can pass along to an MES or similar system.

**Your automation specialist stays in control.** When the agent generates an automation service, it also produces:

- **An integration spec** stating what the PLC will receive, how, and when. Your specialist reviews it and supplies details such as register addresses and deadlines.
- **PLC code** in structured text or ladder logic, which your specialist reviews, tests, and loads.

Agents never load code onto a PLC. Generated PLC code covers only the receiving side of the integration, and it stays out of safety functions such as emergency stops and guarding.

The first supported target is structured text over Modbus TCP, tested on CODESYS and OpenPLC. A simulator won't catch every vendor-specific behavior, so review and on-site testing still matter.

## When something goes wrong

You decide how each inspection fails. An inspection can't be deployed until someone chooses what happens when it can't produce a result, such as letting product pass or stopping the line.

| Problem                   | What the system does                                                                        |
| ------------------------- | ------------------------------------------------------------------------------------------- |
| No result by the deadline | The automation service goes to its configured default state and reports it                  |
| A result arrives late     | The automation service follows the rule set for your PLC, so it can't reject the wrong part |
| A camera misses a trigger | The event ID service detects the mismatch, raises an alert, and resynchronizes the group    |
| A service crashes         | Other inspections continue, and the automation service goes to its default state if needed  |
| A bad update is deployed  | Failed health checks trigger an automatic rollback to the previous version                  |

Line faults reach your PLC and plant systems through the automation service, and operators see them on the HMI, so people on the floor find out without needing an agent.

## Seeing what's happening

Every message carries a microsecond timestamp and an `event_id`. You can follow any inspection through the system and see how long each stage took, from exposure through to the write to the PLC.

There are three ways to look at the system:

- **The status page** runs on the device and needs no internet or agent. It shows service health, each inspection's latest image and result, active faults, and how to connect an agent. It's read-only.
- **The HMI** is the operator screen generated for your inspection. It shows what people at the line need, such as the current result, recent images, counts, and active faults. Operators can also take a few actions, such as acknowledging a fault or marking a part for review. Each action is listed in the HMI's definition file and recorded with who took it and when, against the `event_id` where there is one. Anything that changes line behavior or system settings goes through agents and their permissions instead. The HMI runs on the device with no internet or agent.
- **On-demand views** are created by your AI agent through the client bridge and MCP server, for watching live results, reviewing images, or investigating a problem. A view that works can be saved and reopened.

A control in a generated view can't do anything the agent itself isn't allowed to do.

## Working with AI agents

Agents help in four ways:

| Role     | What the agent does                                                  |
| -------- | -------------------------------------------------------------------- |
| Build    | Asks about your setup, discovers hardware, and generates the system  |
| Monitor  | Watches for missed inspections, model drift, and device health       |
| Maintain | Applies updates and changes services                                 |
| Retrain  | Collects samples, coordinates labeling, trains, and validates models |

### Connecting your client

To work with a running inspection system, you install a small bridge on your AI client. Agents talk to the bridge, and the bridge talks to the MCP server on the inspection system:

```text
Your AI client ──► OpenGadget bridge ──► MCP server ──► Inspection system
(Claude, Codex, …)  (on your computer)  (on the device)
```

The bridge is the one controlled way in. An agent has no direct connection to the device, can use only what the bridge exposes, and never holds the device's credentials. This applies to every agent, including a coding agent deploying from your repository, so each deployment follows the same permissions and approvals. You also aren't tied to one model provider.

The bridge comes from the `mcp-client/` folder as an MCP bundle (MCPB), along with skills for working with a running system.

### Permissions

The MCP server sets what an agent can do according to risk:

| Level      | What an agent can do                                      | Approval needed                 |
| ---------- | --------------------------------------------------------- | ------------------------------- |
| Observe    | Read telemetry, sample images, and view configuration     | None                            |
| Propose    | Draft a pipeline and try it in shadow mode on real images | None                            |
| Deploy     | Promote a pipeline or model, or change retention          | A person                        |
| Automation | Change PLC mappings or failure policies                   | A person, with an on-site check |

A pipeline in shadow mode sees live images but can't reach an automation service, so you can test a change without any risk to the line.

## Repository layout

```text
your-inspection-system/
├── README.md             Start here
├── AGENTS.md             Where agents start: the map, the stages, and the rules
├── CLAUDE.md							Redirects to AGENTS.md
├── .agents/
│   ├── skills/           Guided workflows, such as connecting a sensor or designing a pipeline
│   └── hooks/            Automatic checks while an agent works
├── docs/                 Architecture, risk register, and guides
├── constraints/          What custom services must do
├── conformance/          Tests for those constraints
├── platform/             The base system and the code for building its packages
├── mcp-client/           What you add to your AI client
│   ├── bridge/           The small bridge to the inspection system's MCP server, packaged as an MCPB
│   └── skills/           Skills for monitoring, diagnosing, and maintaining a running system
├── tools/                The command-line tool for building, deploying, testing, and logs
├── examples/             Reference pipelines, automation, and PLC code
└── inspection/           Your system
    ├── system.yaml       Sensors, topics, trigger groups, and failure policies
    ├── sensors/          Configuration for each camera
    ├── pipelines/        One folder per pipeline
    ├── automation/       One folder per service, with its integration spec and PLC code
    ├── hmi/              The operator screen for this inspection
    ├── models/           Model manifests, with weights stored outside Git
    └── tests/            Tests specific to your system
```

**`inspection/` is yours, and everything else comes from OpenGadget.** Agents write code in `inspection/` unless you ask them to change the platform. They can build an inspection using just `AGENTS.md`, the skills, and `constraints/`, without reading the platform's source.

**There are two sets of skills.** `.agents/skills/` helps an agent working in your repository build the system. `mcp-client/skills/` installs with the bridge on your AI client, for working with a system that's already running.

To update the platform, run the update command with the release you want. It replaces the OpenGadget folders and runs the conformance tests against your `inspection/` folder.

## Installing and updating

OpenGadget runs on two kinds of device, and NVIDIA GPUs are the only supported accelerators:

| Platform      | Architecture | Requirements and examples                                 |
| ------------- | ------------ | --------------------------------------------------------- |
| NVIDIA Jetson | ARM64        | Jetson Orin                                               |
| x86 Linux     | x86-64       | An NVIDIA GPU, in an industrial PC or other edge computer |

Your computer builds packages for your device's architecture. There are two Debian packages:

- **The base package** holds the platform services and rarely changes.
- **The inspection package** holds your pipelines, automation, and configuration, and changes as you iterate.

You install the base package on a new device yourself. After that, inspection packages deploy through the bridge and MCP server, so a new pipeline doesn't mean reinstalling the platform. Installing needs no internet access on the device, which suits restricted plant networks, and returning to an earlier version is a simple deployment.

## Technical reference

### Cameras and triggering

OpenGadget supports GigE Vision and GenICam cameras through one generic service, so adding a camera means changing its settings rather than writing code. Software triggers are available for setup and testing. An emulation mode plays test images through the sensor service, so you can test without a camera.

### `event_id` coordination

Cameras that share a trigger form a group, and each group works in numbered rounds so its cameras hold an ID before the trigger fires:

1. Each sensor clears its camera buffers and joins the group.
2. The event ID service publishes a round number and a new `event_id`.
3. Each sensor takes the ID and confirms it's ready. Once all have confirmed, the service generates the next ID.
4. The trigger fires, and each sensor tags its image with the ID it holds.
5. Each sensor moves to the next round.

If a sensor misses a round, the service raises an alert, and the event follows the inspection's failure policy. Camera frame counters provide a second check for dropped images. To recover, the group repeats the first three steps. Coordination covers the sensors on a single device.

### Messages

Messages are JSON, so any service can process them. Each has a schema version, message type, source service, microsecond timestamp, and `event_id`. Image messages also have the image's key and dimensions, and the image itself is never in the message. Other fields are free-form.

Messages that affect an inspection use Valkey Streams, which let a restarted service catch up and allow replay during debugging. Other messages go through the database API to Postgres, so a slow database can't delay an inspection.

### Image storage

Images in process live in a separate Valkey instance from messages. Messages must never be evicted, while images can expire, and separating them keeps large image writes from delaying messages. The data manager saves images to disk according to your policies, such as keeping every defect for 90 days but only a sample of good parts.

### Time

Timestamps are integer microseconds. Wall-clock time records when something happened, and a monotonic clock measures durations, because the wall clock can jump when corrected. Image messages also carry the camera's hardware timestamp when one is available.

### Resources on a device

Parallel inspections share the GPU, memory, image storage, and network bandwidth. Each inspection declares what it needs, and the agent checks that everything fits before deploying. Each physical output belongs to one automation service, and if several pipelines feed it, a configuration rule says how their results combine.

### Models

Labeling and training use existing open source tools and run off the device, on a local GPU or in the cloud. A pipeline asks for a model role, and the model manager determines which model fills that role. Check model licenses before use, since OpenGadget is MIT licensed. For example, Ultralytics YOLO is AGPL-3.0, while YOLOX is permissively licensed.

### Languages

OpenGadget uses the right technology for each component rather than one language throughout:

| Technology | Where it fits                                                                           |
| ---------- | --------------------------------------------------------------------------------------- |
| Rust       | Components close to hardware, or where memory safety and predictable timing matter most |
| Go         | Network services and command-line tooling                                               |
| TypeScript | The status page, HMI, on-demand views, and MCP tooling                                       |
| Python     | Models and pipelines, because most model tooling is in Python                           |

Generated automation services use whichever language suits the machine's protocol and libraries. Services work together regardless of language because messages are JSON and the same conformance tests apply to all of them.

## Risks and open decisions

The [risk register](risks.md) lists what's still unproven or undecided, and how we're handling it.