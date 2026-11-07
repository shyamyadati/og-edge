# Risk Register

These are the risks OpenGadget carries in its current design, including decisions that are still open. Each will be closed by testing or a decision, and this register is updated as that happens. For the design itself, see the [architecture document](architecture.md).

Severity shows how much a risk could affect the project or a production line if it goes wrong: **High**, **Medium**, or **Low**.

## Safety and reliability

| ID | Risk | Severity | What could happen | Current approach |
| --- | --- | --- | --- | --- |
| R1 | Generated automation code misbehaves on a real line | High | Wrong parts are rejected, or a machine behaves unexpectedly | Conformance tests, simulation, approval with an on-site check, and specialist review of PLC code. Agents can't load code onto a PLC. |
| R2 | Generated code is unreliable | High | Systems fail in ways that weren't tested, undermining the project's core promise | A small custom layer, shared base code, conformance tests, and repeated clean builds by agents |
| R3 | Design constraints are too loose or vague | High | Generated services vary and fail on real machines | Test every constraint possible, and regularly build from the constraints alone with a fresh agent |
| R4 | Results become misaligned with physical parts | High | One missed result shifts later results in an order-based PLC queue, so the wrong parts are rejected | An explicit rule for late and missing results in each automation service, covered by conformance tests |
| R5 | Software or timed triggers hide timing problems during testing | Medium | Misalignment appears only on real lines | Test with a true hardware trigger before relying on `event_id` coordination |
| R26 | A generated HMI misleads operators or changes line behavior | High | An unclear screen causes a wrong decision, or an operator control has an unintended effect on the line | Conformance tests for the HMI contract, no system settings on the HMI, and review before deployment. Operators get only the few actions listed in the definition file, each one is recorded, and anything that changes line behavior needs approval. How operators identify themselves is still open. |

## Performance and hardware

| ID | Risk | Severity | What could happen | Current approach |
| --- | --- | --- | --- | --- |
| R6 | Valkey is too slow for images with several high-resolution cameras | Medium | Pipelines are delayed and miss deadlines | Measure on all supported hardware. Memory-mapped files on tmpfs are the fallback, behind the same interface. |
| R7 | Smaller Jetson devices can't hold the full system | High | Too little memory for Valkey, Postgres, images, and models together | Measure early, and offer lighter options or require larger devices |
| R8 | Postgres and image writes wear out storage | Medium | Devices fail early in the field | Measure write volumes, and tune retention and telemetry |
| R9 | Several cameras exceed shared network bandwidth | Medium | Images are dropped during trigger bursts | Test trigger groups, and include bandwidth in device resource checks |
| R24 | Supporting only NVIDIA GPUs limits adoption | Low | People with CPU-only, Intel, or AMD systems can't use OpenGadget, and CI needs a GPU to test real inference | Reassess if there's demand, and run most automated tests with emulated sensors and test pipelines that don't need a GPU |
| R25 | Behavior differs between Jetson and x86 | Medium | A pipeline or model tested on one type of hardware performs differently on the other, or the test burden grows with each type | Hide hardware differences in base images and the model manager, and run conformance and performance tests on both |
| R10 | Inaccurate clocks on plant networks | Low | Performance and traceability data becomes misleading | Use a monotonic clock for durations. Evaluate time synchronization and PTP for cameras. |

## Building and updating

| ID | Risk | Severity | What could happen | Current approach |
| --- | --- | --- | --- | --- |
| R11 | Packages are hard to build for all supported hardware from a typical computer | High | The core install flow doesn't work on a Mac or x86 machine, especially for Jetson GPU libraries | Test early, and decide whether packages contain native binaries, Docker images, or a mix |
| R12 | Platform updates break generated services | Medium | Repositories made from a template don't share Git history with OpenGadget, and an update could change rules that existing services no longer meet | A versioned update command that runs conformance tests against `inspection/`. How it handles failures is still open. |
| R13 | Using several languages increases what we maintain | Medium | Base code must be provided and kept in step across languages | Choose languages per component, keep contracts language-independent, and test all services against the same conformance suite |
| R14 | Agents don't read the standard file locations | Medium | A new repository doesn't load `AGENTS.md` or `.agents/skills/` in some agents, and hooks may need connecting separately for each agent | Test with each agent, and add minimal pointers where needed. The command-line tool and CI enforce the rules regardless of hooks. |

## Agent access

| ID | Risk | Severity | What could happen | Current approach |
| --- | --- | --- | --- | --- |
| R15 | The security design for agent access is incomplete | High | Weak device identity, credentials, or audit logs could allow unauthorized changes to a system | Agents reach the device only through the client bridge, with tiered permissions. All deployments go through the bridge as well. Identity, credentials, and audit logs still need designing. |
| R16 | Not every MCP client can install the bridge | Medium | People using an unsupported or hosted client can't reach their system | Package the bridge as an MCP bundle, and test the clients we intend to support |
| R17 | Generated views can't display or stream live images | Medium | On-demand views are less useful, especially on private plant networks | Test what MCP clients can display, and whether the bridge can carry live images |
| R18 | The status page is too limited during an outage | Medium | A technician can't resolve a problem when no agent is available | It starts as read-only. Decide whether it needs a few controls, such as restarting a service. |
| R19 | People lack telemetry views without an agent | Low | Monitoring is harder for people who don't want to use an agent for everything | Decide whether Grafana is a useful default alongside on-demand views |

## Compatibility and licensing

| ID | Risk | Severity | What could happen | Current approach |
| --- | --- | --- | --- | --- |
| R20 | Licenses conflict with an open source release | High | The terms of the GigE Vision standard, [Aravis](https://github.com/AravisProject/aravis) (LGPL), or a model could restrict how OpenGadget ships | Review them before depending on them, and check every recommended model against this project's MIT license |
| R21 | Support for only GigE Vision and GenICam limits adoption | Medium | 3D and other sensors can't be used | Start with one generic service, and decide later whether and how to add other sensors |
| R22 | Generated PLC code doesn't fit each vendor | Medium | Code needs manual rework, or ladder logic import fails | Start with standard structured text on CODESYS and OpenPLC, then research vendor import formats |
| R23 | Simulators can't fully test PLC integration | Medium | Vendor-specific problems appear only on real machines, and CI coverage may be limited | Use simulators where they work, with specialist review and on-site testing for the rest |
