# FreePBX mit FritzBox verbinden – Vollständige Anleitung

**Ziel:** FreePBX (in Docker auf OMV-Server) vollständig mit der FritzBox koppeln, sodass:
- Eingehende Festnetzanrufe auf gewünschten Telefonen klingeln
- Ausgehende Anrufe (Mobil, Festnetz) über die FritzBox gehen
- FritzBox-interne Nebenstellen (6XX, analoges Telefon) erreichbar sind

***

## Schritt 1: FreePBX als IP-Telefon in der FritzBox einrichten

Die FritzBox kennt FreePBX nicht als Telefonanlage – sie sieht es als normales IP-Telefon. Deshalb wird FreePBX zunächst als Telefoniegerät in der FritzBox angelegt.

1. Öffne die FritzBox-Oberfläche unter [http://fritz.box](http://fritz.box)
2. Gehe zu **Telefonie → Telefoniegeräte → Gerät hinzufügen**
3. Wähle **Telefon (mit und ohne Anrufbeantworter)**
4. Wähle als Anschluss: **LAN/WLAN (IP-Telefon)**
5. Gib dem Gerät einen Namen, z. B. `FreePBX`
6. Lege einen **Benutzernamen** fest – z. B. `FreePBXT`
7. Vergib ein **Passwort** (mind. 8 Zeichen, merken!)
8. Wähle bei **Ausgehende Rufnummer** und **Eingehende Rufnummern** deine Festnetznummer aus
9. Klicke auf **Übernehmen**

> 📌 **Notiere genau:**
> - **Benutzername:** `FreePBXT` ← diesen Wert brauchst du in Schritt 2 an **drei Stellen**
> - **Passwort:** dein gewähltes Passwort
> - **FritzBox-IP:** `192.168.178.1` (Standard)

***

## Schritt 2: PJSIP Trunk in FreePBX anlegen

Jetzt richtest du in FreePBX den Trunk zur FritzBox ein.

1. Gehe zu **Connectivity → Trunks → Add Trunk → Add SIP (chan_pjsip) Trunk**
2. **Trunk Name:** z. B. `FritzBox`

### Reiter: General

| Feld | Wert |
|------|------|
| Trunk Name | `FritzBox` |
| Outbound CallerID | deine Festnetznummer (z. B. `026289XXXXX`) |

### Reiter: pjsip Settings → General

| Feld | Wert |
|------|------|
| Username | `FreePBXT` |
| Secret | dein FritzBox-Passwort |
| SIP Server | `192.168.178.1` |
| SIP Server Port | `5060` |
| Context | `from-trunk` |
| Transport | `UDP` |

### Reiter: pjsip Settings → Advanced

| Feld | Wert | Hinweis |
|------|------|---------|
| **From User** | `FreePBXT` | ⚠️ **Muss exakt dem Benutzernamen aus Schritt 1 entsprechen!** |
| From Domain | `192.168.178.1` | IP der FritzBox |
| Contact User | *(leer lassen)* | Ein Eintrag hier verhindert korrekte eingehende Anrufe |
| Authentication | `Outbound` | Nicht „Both" – sonst schlagen eingehende Anrufe fehl (Fehler 408) |

> ⚠️ **Kritisch: `Username` und `From User` müssen identisch sein und exakt dem Benutzernamen entsprechen, den du in der FritzBox unter Telefoniegeräte vergeben hast.**
> Stimmen diese nicht überein, lehnt die FritzBox die Registrierung ab oder ausgehende Anrufe schlagen mit „403 Forbidden" fehl.

3. Klicke auf **Submit**, dann oben auf **Apply Config**

### Registrierung prüfen

In der Asterisk-CLI:

```bash
asterisk -rx "pjsip show registrations"
```

Der Status neben dem FritzBox-Trunk sollte `Registered` zeigen.

***

## Schritt 3: Inbound Route einrichten

Die Inbound Route legt fest, was mit eingehenden Anrufen passiert (wer klingelt).

1. Gehe zu **Connectivity → Inbound Routes → Add Inbound Route**
2. Lass alle Felder bei Standardwerten (kein DID/CID nötig bei FritzBox-Trunk)
3. Setze als **Destination** eine **Ring Group** (empfohlen, siehe unten)

### Ring Group anlegen

Damit mehrere Telefone gleichzeitig klingeln:

1. Gehe zu **Applications → Ring Groups → Add Ring Group**
2. Wähle einen Namen, z. B. `Alle Telefone`
3. Füge unter **Extension List** alle gewünschten Extensions ein (z. B. `4434`, weitere SIP-Telefone)
4. Stelle **Ring Strategy** auf `ringall`
5. Setze einen Timeout (z. B. `30` Sekunden)
6. Als **Destination if no answer** kannst du Voicemail oder Hangup wählen
7. **Submit** → **Apply Config**

Zurück in der Inbound Route: als Destination nun diese Ring Group auswählen.

***

## Schritt 4: Outbound Routes einrichten

Outbound Routes bestimmen, welche gewählten Nummern über welchen Trunk geleitet werden.

### Route: Extern (Festnetz & Mobil)

1. Gehe zu **Connectivity → Outbound Routes → Add Outbound Route**
2. **Route Name:** z. B. `Extern-FritzBox`
3. **Trunk Sequence:** deinen `FritzBox`-Trunk auswählen

#### Dial Patterns

Trage folgende Patterns ein (je eine Zeile):

| Prepend | Prefix | Match Pattern | Erklärung |
|---------|--------|---------------|-----------|
| *(leer)* | *(leer)* | `0XXXXXXXXXX.` | Deutsche Festnetz- & Mobilnummern |
| *(leer)* | *(leer)* | `00X.` | Internationale Nummern |

> Die FritzBox erwartet Nummern im normalen deutschen Format (z. B. `017648729519`). Kein `**`-Präfix nötig – das wird nur für FritzBox-interne Nummern in Schritt 5 verwendet.

4. **Submit** → **Apply Config**

***

## Schritt 5: FritzBox-interne Nebenstellen erreichbar machen

FritzBox-interne Nummern (Nebenstellen `6XX`, analoges Telefon `FON 1`) sind nicht direkt über den normalen Dial-Plan erreichbar, weil FreePBX das `**`-Präfix intern als Feature Code (Call Pickup) interpretiert – und `**620` daher nie den Trunk erreicht.

Die Lösung: ein **Custom Dialplan**, der das `**` erst unmittelbar beim Dial-Befehl einfügt.

### Datei bearbeiten

Öffne auf dem Server:

```
/etc/asterisk/extensions_custom.conf
```

Füge folgenden Block ein (oder ergänze `[from-internal-custom]`, falls er schon existiert):

```ini
[from-internal-custom]

; FritzBox interne Nebenstellen (6XX) -> sendet **6XX an FritzBox
exten => _6XX,1,Dial(PJSIP/**${EXTEN}@FritzBox,30)
exten => _6XX,n,Hangup()

; FritzBox analoges Telefon FON 1 -> sendet **1 an FritzBox
exten => 1,1,Dial(PJSIP/**1@FritzBox,30)
exten => 1,n,Hangup()

; FritzBox analoges Telefon FON 2 (falls vorhanden)
exten => 2,1,Dial(PJSIP/**2@FritzBox,30)
exten => 2,n,Hangup()
```

**Erklärung jeder Zeile:**

| Element | Bedeutung |
|---------|-----------|
| `_6XX` | Passt auf alle dreistelligen Nummern beginnend mit 6 (z. B. 620, 663) |
| `**${EXTEN}` | Setzt `**` direkt vor die gewählte Nummer (z. B. → `**620`) |
| `@FritzBox` | Name des Trunks, wie in FreePBX angelegt |
| `30` | Timeout in Sekunden – nach 30 s ohne Abnahme wird aufgelegt |
| `n` | Nächster Schritt: Hangup, falls Dial fehlschlägt |

### Dialplan neu laden

```bash
asterisk -rx "dialplan reload"
```

Kein Neustart nötig – der Dialplan ist sofort aktiv.

### Testen

```bash
asterisk -rx "dialplan show 620@from-internal"
```

Der Custom-Eintrag sollte dort erscheinen. Dann vom Telefon `620` wählen – die FritzBox-Nebenstelle klingelt.

***

## Schnellreferenz: FritzBox-interne Nummern

| Wählen | Sendet an FritzBox | Gerät |
|--------|-------------------|-------|
| `620` | `**620` | Nebenstelle 620 |
| `621` | `**621` | Nebenstelle 621 |
| `1` | `**1` | FON 1 (analoges Telefon) |
| `2` | `**2` | FON 2 (analoges Telefon) |

***

## Troubleshooting

### Trunk registriert sich nicht / 403 Forbidden

- Prüfe, ob **Username**, **From User** und der **Benutzername in der FritzBox** exakt übereinstimmen (Groß-/Kleinschreibung beachten!)
- Stelle sicher, dass `Authentication = Outbound` gesetzt ist
- Prüfe Erreichbarkeit: `ping 192.168.178.1`
- Registrierungsstatus: `asterisk -rx "pjsip show registrations"`

### Eingehende Anrufe klingeln überall / nirgends

- Prüfe die Inbound Route: sie sollte auf die korrekte Ring Group zeigen
- Ring Group: enthält sie alle gewünschten Extensions?
- Nach jeder Änderung: **Apply Config** klicken

### Interne FritzBox-Nummern nicht erreichbar

- Prüfe ob `extensions_custom.conf` korrekt gespeichert ist
- `asterisk -rx "dialplan reload"` ausführen
- Testen: `asterisk -rx "dialplan show 620@from-internal"`

### Ausgehende Anrufe schlagen fehl

- Prüfe die Outbound Route und Dial Patterns
- Beobachte den Anruf live: `asterisk -rvvv` und dann wählen

***

*Erstellt: April 2026 | Konfiguration: FreePBX auf Docker/OMV · FritzBox als SIP-Trunk*
