# Operations

This document explains how an OpenGadget system handles problems, how you see what it's doing, and how you install and update it.

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

## Installing and updating

Your computer builds packages for your device's architecture. There are two Debian packages:

- **The platform package** holds the [platform](platform.md) services and rarely changes.
- **The inspection package** holds your pipelines, automation, and configuration, and changes as you iterate.

You install the platform package on a new device yourself. After that, inspection packages deploy through the bridge and MCP server, so a new pipeline doesn't mean reinstalling the platform. Installing needs no internet access on the device, which suits restricted plant networks, and returning to an earlier version is a simple deployment.
