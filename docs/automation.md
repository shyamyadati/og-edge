# Automation

An automation service connects the inspection system to your PLCs and automation equipment. These are generated because every machine integration is different, and you can regenerate one quickly when a machine changes.

**Your PLC stays in charge of tracking product.** Whether your line uses fixed delays, specific registers, or queues inside the PLC, OpenGadget writes results where the PLC expects them. It can also write `event_id`, which your PLC can pass along to an MES or similar system.

**Your automation specialist stays in control.** When the agent generates an automation service, it also produces:

- **An integration spec** stating what the PLC will receive, how, and when. Your specialist reviews it and supplies details such as register addresses and deadlines.
- **PLC code** in structured text or ladder logic, which your specialist reviews, tests, and loads.

Agents never load code onto a PLC. Generated PLC code covers only the receiving side of the integration, and it stays out of safety functions such as emergency stops and guarding.

The first supported target is structured text over Modbus TCP, tested on CODESYS and OpenPLC. A simulator won't catch every vendor-specific behavior, so review and on-site testing still matter.
