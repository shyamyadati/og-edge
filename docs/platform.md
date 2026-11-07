# Platform

The platform is the set of services and shared code that OpenGadget installs on every device. It's the same for every inspection system: you configure it, and your generated services run on top of it. For how these services work together, see the [architecture](architecture.md).

## Services

| Service          | What it does for you                                                                |
| ---------------- | ----------------------------------------------------------------------------------- |
| Valkey           | Carries messages and holds images in memory while they're being processed           |
| Postgres         | Stores results, telemetry, configuration, and audit logs                            |
| Database API     | Gives services a common way to use Postgres, so generated code doesn't need SQL     |
| Sensor service   | Captures from any GigE Vision or GenICam camera, configured by the agent            |
| Event ID service | Keeps `event_id` in step across cameras that share a trigger                        |
| Data manager     | Archives images to disk and prunes old data according to your retention rules       |
| Model manager    | Tracks which model is in use for each role and prepares models to run on the device |
| System monitor   | Reports on storage, memory, GPU, and other device health                            |
| Logging service  | Records telemetry and status so you can trace what happened                         |
| MCP server       | Lets AI agents inspect and manage the device, within the permissions you set        |
| Status page      | Shows device and inspection status to people on site, with no agent or internet     |

Generated services run in containers, so a faulty one can't take down the platform. A prebuilt image for each supported type of hardware supplies the drivers and inference runtime for pipelines, so a pipeline doesn't have to handle hardware differences itself.

## Where it lives

```text
platform/
├── services/     The services listed above
├── libs/         The layered base classes used by every service
├── infra/        Valkey and Postgres configuration, and the database schema
└── packaging/    Debian package builds
```

## Installing and updating

The platform ships as its own Debian package, separate from your inspection package. See [Operations](operations.md) for how to install and update it.

Agents building an inspection system don't need to read the platform's source. To change the platform itself, work in `platform/`, and only when someone asks for it.
