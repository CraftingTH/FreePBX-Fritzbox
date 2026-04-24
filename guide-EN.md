# FreePBX + FritzBox – Complete Setup Guide

**Goal:** Fully integrate FreePBX (running in Docker on an OMV server) with a FritzBox router so that:
- Incoming landline calls ring on the desired phones
- Outgoing calls (mobile, landline) are routed through the FritzBox
- FritzBox-internal extensions (6XX, analog phone) are reachable from FreePBX

***

## Step 1: Add FreePBX as an IP Phone in the FritzBox

The FritzBox does not recognize FreePBX as a PBX — it treats it like a regular IP phone. Therefore, FreePBX must first be registered as a telephony device in the FritzBox.

1. Open the FritzBox web interface at [http://fritz.box](http://fritz.box)
2. Navigate to **Telephony → Telephony Devices → Add Device**
3. Select **Phone (with or without answering machine)**
4. Select connection type: **LAN/WLAN (IP Phone)**
5. Give it a name, e.g. `FreePBX`
6. Set a **username** – e.g. `FreePBXT`
7. Set a **password** (min. 8 characters – write it down!)
8. Under **Outgoing number** and **Incoming numbers**, select your landline number
9. Click **Apply**

> 📌 **Note down carefully:**
> - **Username:** `FreePBXT` ← you will need this at **three places** in Step 2
> - **Password:** your chosen password
> - **FritzBox IP:** `192.168.178.1` (default)

***

## Step 2: Create a PJSIP Trunk in FreePBX

Now configure the trunk in FreePBX that connects to the FritzBox.

1. Go to **Connectivity → Trunks → Add Trunk → Add SIP (chan_pjsip) Trunk**
2. **Trunk Name:** e.g. `FritzBox`

### Tab: General

| Field | Value |
|-------|-------|
| Trunk Name | `FritzBox` |
| Outbound CallerID | your landline number (e.g. `026289XXXXX`) |

### Tab: pjsip Settings → General

| Field | Value |
|-------|-------|
| Username | `FreePBXT` |
| Secret | your FritzBox password |
| SIP Server | `192.168.178.1` |
| SIP Server Port | `5060` |
| Context | `from-trunk` |
| Transport | `UDP` |

### Tab: pjsip Settings → Advanced

| Field | Value | Note |
|-------|-------|------|
| **From User** | `FreePBXT` | ⚠️ **Must exactly match the username set in Step 1!** |
| From Domain | `192.168.178.1` | FritzBox IP address |
| Contact User | *(leave empty)* | Any value here will break incoming calls |
| Authentication | `Outbound` | Do NOT use „Both" – causes incoming call failures (error 408) |

> ⚠️ **Critical: `Username` and `From User` must be identical and must exactly match the username you set in the FritzBox in Step 1.**
> If they differ, the FritzBox will reject the registration or outgoing calls will fail with „403 Forbidden".

3. Click **Submit**, then **Apply Config** at the top

### Verify Registration

In the Asterisk CLI:

```bash
asterisk -rx "pjsip show registrations"
```

The status next to the FritzBox trunk should show `Registered`.

***

## Step 3: Set Up an Inbound Route

The inbound route defines what happens with incoming calls (which phones ring).

1. Go to **Connectivity → Inbound Routes → Add Inbound Route**
2. Leave all fields at defaults (no DID/CID needed for FritzBox trunk)
3. Set **Destination** to a **Ring Group** (recommended, see below)

### Create a Ring Group

To make multiple phones ring simultaneously:

1. Go to **Applications → Ring Groups → Add Ring Group**
2. Choose a name, e.g. `All Phones`
3. Under **Extension List**, add all desired extensions (e.g. `4434`, other SIP phones)
4. Set **Ring Strategy** to `ringall`
5. Set a timeout (e.g. `30` seconds)
6. Set **Destination if no answer** to voicemail or hangup
7. **Submit** → **Apply Config**

Go back to the Inbound Route and select this Ring Group as the destination.

***

## Step 4: Set Up Outbound Routes

Outbound routes determine which dialed numbers are sent through which trunk.

### Route: External (Landline & Mobile)

1. Go to **Connectivity → Outbound Routes → Add Outbound Route**
2. **Route Name:** e.g. `External-FritzBox`
3. **Trunk Sequence:** select your `FritzBox` trunk

#### Dial Patterns

Enter the following patterns (one per line):

| Prepend | Prefix | Match Pattern | Description |
|---------|--------|---------------|-------------|
| *(empty)* | *(empty)* | `0XXXXXXXXXX.` | German landline & mobile numbers |
| *(empty)* | *(empty)* | `00X.` | International numbers |

> The FritzBox expects numbers in standard German format (e.g. `017648729519`). No `**` prefix is needed here — that is only used for FritzBox-internal numbers in Step 5.

4. **Submit** → **Apply Config**

***

## Step 5: Reach FritzBox-Internal Extensions

FritzBox-internal extensions (e.g. `6XX`, analog phone `FON 1`) are not reachable through the normal FreePBX dial plan, because FreePBX interprets the `**` prefix as a Feature Code (Call Pickup) — so `**620` never reaches the trunk.

The solution: a **custom dial plan** that inserts `**` directly at the `Dial()` command, bypassing FreePBX's internal routing.

### Edit the File

Open on the server:

```
/etc/asterisk/extensions_custom.conf
```

Add the following block (or extend `[from-internal-custom]` if it already exists):

```ini
[from-internal-custom]

; FritzBox internal extensions (6XX) -> sends **6XX to FritzBox
exten => _6XX,1,Dial(PJSIP/**${EXTEN}@FritzBox,30)
exten => _6XX,n,Hangup()

; FritzBox analog phone FON 1 -> sends **1 to FritzBox
exten => 1,1,Dial(PJSIP/**1@FritzBox,30)
exten => 1,n,Hangup()

; FritzBox analog phone FON 2 (if available)
exten => 2,1,Dial(PJSIP/**2@FritzBox,30)
exten => 2,n,Hangup()
```

**Line-by-line explanation:**

| Element | Meaning |
|---------|---------|
| `_6XX` | Matches any three-digit number starting with 6 (e.g. 620, 663) |
| `**${EXTEN}` | Prepends `**` to the dialed number (e.g. → `**620`) |
| `@FritzBox` | Trunk name as configured in FreePBX |
| `30` | Timeout in seconds – hangs up after 30 s if no answer |
| `n` | Next priority: Hangup if Dial fails |

### Reload the Dial Plan

```bash
asterisk -rx "dialplan reload"
```

No restart required — changes take effect immediately.

### Test

```bash
asterisk -rx "dialplan show 620@from-internal"
```

The custom entry should appear. Then dial `620` from a phone — the FritzBox extension should ring.

***

## Quick Reference: FritzBox-Internal Numbers

| Dial | Sent to FritzBox | Device |
|------|-----------------|--------|
| `620` | `**620` | Extension 620 |
| `621` | `**621` | Extension 621 |
| `1` | `**1` | FON 1 (analog phone) |
| `2` | `**2` | FON 2 (analog phone) |

***

## Troubleshooting

### Trunk not registering / 403 Forbidden

- Check that **Username**, **From User**, and the **FritzBox username from Step 1** are exactly identical (case-sensitive!)
- Ensure `Authentication = Outbound` is set
- Check connectivity: `ping 192.168.178.1`
- Check registration status: `asterisk -rx "pjsip show registrations"`

### Incoming calls ring everywhere / nowhere

- Check the Inbound Route: it should point to the correct Ring Group
- Verify the Ring Group contains all desired extensions
- Always click **Apply Config** after any change

### FritzBox-internal numbers not reachable

- Verify `extensions_custom.conf` is saved correctly
- Run `asterisk -rx "dialplan reload"`
- Test: `asterisk -rx "dialplan show 620@from-internal"`

### Outgoing calls failing

- Check the Outbound Route and Dial Patterns
- Monitor a live call: `asterisk -rvvv`, then dial and observe the output

***

*Created: April 2026 | Setup: FreePBX 17.0.28 | Asterisk 22.9.0 on Docker/OMV · FritzBox as SIP trunk*
