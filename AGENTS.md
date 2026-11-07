# OpenGadget

OpenGadget is an open source visual inspection system for production lines. A person works with an AI coding agent in their own repository to build, test, and maintain an inspection system that runs on an NVIDIA Jetson or an x86 Linux computer with an NVIDIA GPU.

## Current state

The project is in early design, and only documentation exists so far. The README and docs describe the target design, including folders and tools that haven't been created yet. Don't assume something exists because a document mentions it, and check the risk register before treating a design detail as final.

## Where to look

| To understand                                                  | Read                                                 |
| -------------------------------------------------------------- | ---------------------------------------------------- |
| What the project is, the build stages, and the repository layout | [README.md](README.md)                               |
| How the running system works                                   | [docs/architecture.md](docs/architecture.md)         |
| The services and shared code on every device                  | [docs/platform.md](docs/platform.md)                 |
| How custom services are built and tested                       | [docs/custom-services.md](docs/custom-services.md)   |
| PLC and machine connections                                    | [docs/automation.md](docs/automation.md)             |
| Agent roles, the client bridge, and permissions                | [docs/agents.md](docs/agents.md)                     |
| Failures, monitoring, installing, and updating                 | [docs/operations.md](docs/operations.md)             |
| What's unproven or undecided                                   | [docs/risks.md](docs/risks.md)                       |

Read only what your task needs.

## Design rules

Keep to these rules unless a person explicitly changes them. If a change affects one, update the relevant document in the same piece of work.

- **Deterministic on the device:** No LLM runs on the device or in the inspection path. Inspection must keep running if an agent or the network goes away.
- **Regular services:** Every service uses the same layers of messages, interfaces, and services. New abstractions must make services more regular and faster to build.
- **Generate only what's custom:** Common work belongs in the platform. Generated code covers only what's specific to an inspection or machine.
- **Tests before deployment:** A generated service must pass conformance tests before it can deploy. Wherever possible, each design constraint has a test.
- **One way in:** Agents reach a device only through the client bridge and MCP server, including for deployments. Installing the platform package on a new device is the only manual exception.
- **No PLC loading:** Agents never load code onto a PLC. Automation specialists review, test, and load generated PLC code, and it stays out of safety functions.
- **Right technology per component:** Choose Rust, Go, TypeScript, or Python according to what each component needs.
- **Supported hardware:** NVIDIA GPUs are the only supported accelerators, on Jetson (ARM64) and x86-64 Linux.

## Working in this repository

- **Terms:** The platform is the services and shared code on every device, in `platform/`. Use "hardware" for Jetson and x86, not "platform".
- **Ownership:** In a person's copy of the template, `inspection/` belongs to them and everything else comes from OpenGadget. Generate code only in `inspection/` unless someone asks you to change the platform.
- **Standard agent files:** Put agent instructions and skills in `AGENTS.md` and `.agents/`. Files for a specific agent contain only a pointer to them.
- **Documentation:** Write for the reader, and describe only what this repository implements or is building toward. Don't add project history, retrospectives, or comparisons with other systems.
- **Risks:** Record open decisions and unproven assumptions in [docs/risks.md](docs/risks.md), and update an entry when it changes.
- **Secrets:** Never commit credentials, device addresses, or customer data.
- **Local notes:** `.tmp/` is git-ignored and may hold private notes. Don't read it unless asked, and don't copy from it into tracked files.
