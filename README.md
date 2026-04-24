# FreePBX + FritzBox Integration

This repository provides a complete, step-by-step guide to integrating **FreePBX** (running in Docker on an OMV server) with an **AVM FritzBox** router as a SIP trunk — enabling inbound/outbound landline calls and access to FritzBox-internal extensions.

## Guides

| Language | Link |
|----------|------|
| 🇬🇧 English | [guide-EN.md](guide-EN.md) |
| 🇩🇪 Deutsch | [guide-DE.md](guide-DE.md) |

## What's covered

- Registering FreePBX as an IP phone in the FritzBox
- Setting up a PJSIP trunk in FreePBX
- Routing inbound calls to a ring group
- Configuring outbound routes for landline and mobile calls
- Reaching FritzBox-internal extensions via a custom dial plan

---

*Tested on: FreePBX / Asterisk 22 · AVM FritzBox · Docker on OpenMediaVault*
