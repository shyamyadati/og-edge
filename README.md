# OpenGadget

## Vision

OpenGadget helps you build a visual inspection system for a production line by working with an AI coding agent. You describe your cameras, your machines, and what you want to inspect. The agent configures the prepackaged parts, writes the custom code, and tests it with you. The result runs on an NVIDIA Jetson or an x86 Linux computer with an NVIDIA GPU, and keeps running with no AI involved.

> **Project status:** OpenGadget is in early design, and nothing is built yet. These documents describe what we're building toward, and details may change. See the [risk register](docs/risks.md) for open decisions and risks.

## What makes it different

- **You work with an agent instead of a configuration screen.** The agent does the setup and coding in your own repository, and you review the results at each step.
- **The agent only writes what's specific to you.** Camera capture, messaging, storage, and monitoring come ready to use. The agent writes your inspection logic and the connection to your machines.
- **Your line never depends on AI.** Only ordinary, deterministic software runs on the device. If the agent, the network, or the model provider goes away, inspection keeps running.
- **Generated code has to pass tests.** A service can't be deployed until it passes tests covering its messages, health reporting, and failure behavior.
- **You can run many inspections on one device.** A single device runs multiple cameras, pipelines, and machine connections in parallel.

## Supported hardware

OpenGadget runs on two kinds of device, and NVIDIA GPUs are the only supported accelerators:

| Hardware      | Architecture | Requirements and examples                                 |
| ------------- | ------------ | --------------------------------------------------------- |
| NVIDIA Jetson | ARM64        | Jetson Orin                                               |
| x86 Linux     | x86-64       | An NVIDIA GPU, in an industrial PC or other edge computer |

## Building an inspection system

1. **Create your repository.** Use the OpenGadget GitHub template. The new repository is yours, and it can be private.
2. **Start your agent.** Open an AI coding agent such as Claude or Codex in the repository and describe your setup. For example: "Let's build an inspection system that connects to a GigE Vision camera."
3. **Build and install.** Build a Debian package on your computer, copy it to your inspection device, and start the system.
4. **Build up in stages.** Add one piece at a time with your agent, checking each one before you move on.

| Stage         | What you do with the agent                           | How you know it worked                                   |
| ------------- | ---------------------------------------------------- | -------------------------------------------------------- |
| 1. Platform   | Install and start it on the device                   | The device's status page shows healthy services          |
| 2. Sensor     | Discover and configure your camera                   | Triggered images appear                                  |
| 3. Data       | Collect sample images                                | You have a usable set of good and bad parts              |
| 4. Pipeline   | Generate and test the inspection                     | Results are correct on known samples and live images     |
| 5. Automation | Generate the machine connection and PLC code         | Results arrive correctly at your PLC                     |
| 6. HMI        | Generate the operator screen for the inspection      | Operators can see results and faults at the line         |
| 7. Expand     | Add more cameras, pipelines, and machine connections | Everything passes together, including failure tests      |
| 8. Operate    | Monitor, maintain, and retrain                       | Problems get found and fixed through changes you approve |

Your repository holds your configuration, code, PLC integration details, and history, so you or a future agent can see how the system got to its current state and roll back a change.

## Repository layout

```text
your-inspection-system/
├── README.md             Start here
├── AGENTS.md             Where agents start: the map, the stages, and the rules
├── CLAUDE.md             Redirects to AGENTS.md
├── .agents/
│   ├── skills/           Guided workflows, such as connecting a sensor or designing a pipeline
│   └── hooks/            Automatic checks while an agent works
├── docs/                 Architecture, risk register, and guides
├── constraints/          What custom services must do
├── conformance/          Tests for those constraints
├── platform/             The services and shared code on every device, and the code for building its packages
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

## Documentation

| Document                                   | What it covers                                                       |
| ------------------------------------------ | -------------------------------------------------------------------- |
| [Architecture](docs/architecture.md)       | How the running system works, its services, and its data flow        |
| [Platform](docs/platform.md)               | The services and shared code on every device                         |
| [Custom services](docs/custom-services.md) | How pipelines and other services are built, tested, and given models |
| [Automation](docs/automation.md)           | Connecting to PLCs and automation equipment                          |
| [AI agents](docs/agents.md)                | What agents do, the client bridge, and permissions                   |
| [Operations](docs/operations.md)           | Failures, monitoring, installing, and updating                       |
| [Risk register](docs/risks.md)             | What's still unproven or undecided                                   |

## License

OpenGadget is available under the [MIT License](LICENSE).