# AI Agents

This document explains what agents do in an OpenGadget system and how their access is limited.

Agents help in four ways:

| Role     | What the agent does                                                  |
| -------- | -------------------------------------------------------------------- |
| Build    | Asks about your setup, discovers hardware, and generates the system  |
| Monitor  | Watches for missed inspections, model drift, and device health       |
| Maintain | Applies updates and changes services                                 |
| Retrain  | Collects samples, coordinates labeling, trains, and validates models |

## Connecting your client

To work with a running inspection system, you install a small bridge on your AI client. Agents talk to the bridge, and the bridge talks to the MCP server on the inspection system:

```text
Your AI client ──► OpenGadget bridge ──► MCP server ──► Inspection system
(Claude, Codex, …)  (on your computer)  (on the device)
```

The bridge is the one controlled way in. An agent has no direct connection to the device, can use only what the bridge exposes, and never holds the device's credentials. This applies to every agent, including a coding agent deploying from your repository, so each deployment follows the same permissions and approvals. You also aren't tied to one model provider.

The bridge comes from the `mcp-client/` folder as an MCP bundle (MCPB), along with skills for working with a running system.

## Permissions

The MCP server sets what an agent can do according to risk:

| Level      | What an agent can do                                      | Approval needed                 |
| ---------- | --------------------------------------------------------- | ------------------------------- |
| Observe    | Read telemetry, sample images, and view configuration     | None                            |
| Propose    | Draft a pipeline and try it in shadow mode on real images | None                            |
| Deploy     | Promote a pipeline or model, or change retention          | A person                        |
| Automation | Change PLC mappings or failure policies                   | A person, with an on-site check |

A pipeline in shadow mode sees live images but can't reach an automation service, so you can test a change without any risk to the line.
