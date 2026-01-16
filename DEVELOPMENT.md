# Development Guide - Home Assistant Konfiguration

> **Zweck**: Kritisches Wissen für alle Entwickler (Menschen & Claude Code).
> **Aktualisiert**: 2025-01-16

## Projekt-Übersicht

**Was macht dieses Projekt?**
Home Assistant Konfiguration für ein Einfamilienhaus mit Fokus auf Energiemanagement. Das System integriert PV-Anlage, Batteriespeicher (Viessmann E3/Vitocharge), Wallbox (Webasto Unite), E-Auto (Kia Niro EV) und gridX Cloud-Daten zur intelligenten Steuerung des Energieflusses.

**Haupttechnologien**
- Home Assistant (YAML-Konfiguration)
- REST API Integration (gridX API)
- Modbus TCP (Wallbox Webasto Unite)
- Homematic (CCU2 für Hausautomation)
- Jinja2 Templates

**Projektstruktur**
```
homeassistant/
├── configuration.yaml      # Hauptkonfiguration (NICHT DIREKT BEARBEITEN für neue Features)
├── integrations/           # Package-Ordner für modulare Integrationen
│   └── gridx.yaml          # gridX REST Integration (aktiv)
├── modbus.yaml             # Webasto Unite Wallbox Modbus-Register
├── modbus2.yaml            # Erweiterte Modbus-Konfiguration
├── scripts.yaml            # Automations-Scripts (z.B. E-Auto Ladung)
├── secrets.yaml            # Credentials (⚠️ NIEMALS COMMITTEN mit echten Daten)
├── scenes.yaml             # Szenen (leer)
├── gridx.yaml              # ⚠️ VERALTET - nicht mehr verwendet
├── rest.yaml               # ⚠️ VERALTET - nicht mehr verwendet
└── gridX Integration Zusammenfassung.pdf  # Dokumentation der gridX-Versuche
```

**Lokales Setup**
```bash
# 1. Repository klonen
git clone https://github.com/nero2465/homeassistant.git
cd homeassistant

# 2. Für Tests: Home Assistant Developer Tools verwenden
#    - YAML-Syntax prüfen: Developer Tools → YAML → Check Configuration
#    - Templates testen: Developer Tools → Template

# 3. Secrets anpassen (secrets.yaml)
#    - gridx_token: Eigener Bearer Token
#    - gridx_system_id: Eigene System-ID
```

---

## ⚠️ Kritische Erkenntnisse & Known Issues

### gridX API Integration - Template-Engine Limitierungen

**Problem**: REST-Sensor zeigt "unknown" trotz korrekter API-Antwort
**Datum**: 30.12.2025

**❌ Gescheiterte Ansätze**:
- `sensor: platform: rest` mit `json_attributes: true` → State bleibt "unknown"
- `value_template` mit Array-Zugriff `batteries[0].stateOfCharge` → Template-Engine kann keine komplexe Indizierung
- `hass.execute_service()` in Templates → Services sind asynchron, Templates können nicht warten
- HACS `hacs.get_json()` → Funktion nicht in Standard-HA verfügbar

**✅ Funktionierende Lösung**:
- Moderne `rest:` Syntax (Root-Level, nicht unter `sensor:`)
- JSON-Daten als Attribute im Raw-Sensor speichern
- Template-Sensoren lesen aus Entity-Attributen (nicht direkt aus API)
- `Commit: aa2ed58`
- `Datei: integrations/gridx.yaml`

**Wichtig zu beachten**:
- Token läuft nach ~24h ab → Manuelles Erneuern erforderlich
- Verschachtelte Arrays (wie `batteries[0]`) nur über Template-Sensoren zugänglich
- Immer `availability` Template definieren für robuste Sensoren

### gridX Token-Management

**Problem**: OAuth2 Token läuft alle 24h ab
**Status**: Manueller Workaround aktiv

**Workaround**:
1. Login auf https://eon.gridx.de/live-view
2. Browser DevTools → Network → Request Header kopieren
3. Token in `secrets.yaml` aktualisieren

**Geplante Lösung**: AppDaemon mit automatischem Token-Refresh (siehe PDF-Dokumentation)

---

## Architektur-Entscheidungen

### Package-System für modulare Integrationen

**Entscheidung**: Neue Integrationen in `integrations/` Ordner statt in `configuration.yaml`
**Grund**: Bessere Übersichtlichkeit, einfachere Wartung, weniger Merge-Konflikte
**Datum**: 31.12.2025
**Umsetzung**: `packages: !include_dir_named integrations` in configuration.yaml

### REST vs. AppDaemon für externe APIs

**Entscheidung**: Erst REST probieren, bei Komplexität auf AppDaemon wechseln
**Grund**: REST ist einfacher zu warten, AppDaemon erfordert Python-Kenntnisse
**Alternativen**:
- HACS Custom Integration (nicht gefunden für gridX)
- Node-RED (zusätzliche Komplexität)

---

## Code-Patterns & Konventionen

### Datei-Namenskonventionen
- Kleinschreibung mit Unterstrichen: `gridx.yaml`, `modbus2.yaml`
- Integrations-Pakete: `integrations/{integration_name}.yaml`

### Sensor-Definitionen
```yaml
# Immer unique_id verwenden!
- name: "Sensor Name"
  unique_id: sensor_name_lowercase
  unit_of_measurement: "W"
  device_class: power
  state_class: measurement
```

### Secrets-Referenzen
```yaml
# Credentials IMMER über !secret
Authorization: !secret gridx_token

# NIEMALS hardcoded Tokens in YAML-Dateien!
```

### Template-Sensoren mit Availability
```yaml
# Immer availability prüfen für externe Datenquellen
availability: >-
  {{ state_attr('sensor.source', 'attribute') is not none }}
state: >-
  {{ state_attr('sensor.source', 'attribute') | float(0) }}
```

### Kommentare in YAML
```yaml
# =============================================================================
# Abschnitts-Header für große Blöcke
# =============================================================================

# -----------------------------------------------------------------------------
# Unter-Abschnitte
# -----------------------------------------------------------------------------

# Inline-Kommentare für einzelne Zeilen
```

---

## Testing-Hinweise

**Test-Setup**:
- Home Assistant Developer Tools → YAML → Check Configuration
- Home Assistant Developer Tools → Template (für Jinja2 Tests)
- Home Assistant Developer Tools → States (für Sensor-Werte)

**Kritische Test-Szenarien**:
1. **Nach YAML-Änderungen**: Immer "Check Configuration" ausführen
2. **Neue Sensoren**: In States prüfen ob sie erscheinen und Werte haben
3. **Template-Sensoren**: Im Template-Editor vorher testen
4. **Modbus-Register**: Mit Modbus-Tool (z.B. QModMaster) verifizieren

**Bekannte Test-Limitierungen**:
- gridX API nur mit gültigem Token testbar
- Modbus nur mit angeschlossener Wallbox
- Einige Sensoren benötigen echte Hardware-Daten

---

## Nächste Schritte & Offene Punkte

- [ ] gridX Token automatisch refreshen (AppDaemon oder Automation)
- [ ] OAuth2 Flow für gridX implementieren
- [ ] Dashboard für gridX-Daten erstellen
- [ ] Strompreis-Integration (aWATTar, Tibber) hinzufügen
- [ ] Alte Dateien aufräumen (gridx.yaml, rest.yaml im Root)

---

## Für Claude Code User

**Session-Start Ritual**:
```bash
# 1. Repository aktualisieren
git pull

# 2. Kontext laden
cat DEVELOPMENT.md

# 3. Letzte Commits prüfen
git log --oneline -10

# 4. Aktuelle Branch prüfen
git branch -v
```

**Wichtige Dateien zum Lesen bei Änderungen**:

| Änderungstyp | Zuerst lesen |
|--------------|--------------|
| Neue Integration | `configuration.yaml`, `integrations/` |
| Sensor hinzufügen | `configuration.yaml` (template: Abschnitt) |
| Modbus-Änderung | `modbus.yaml`, `modbus2.yaml` |
| Credentials | `secrets.yaml` (⚠️ nicht committen!) |
| gridX spezifisch | `integrations/gridx.yaml`, PDF-Dokumentation |

**Commit-Konventionen**:
```
feat: Neue Funktion hinzugefügt
fix: Bug behoben
docs: Dokumentation aktualisiert
refactor: Code umstrukturiert ohne Funktionsänderung
```

**Vor jedem Push prüfen**:
1. Keine echten Credentials in secrets.yaml committed?
2. YAML-Syntax valide? (`yamllint` oder HA Check Configuration)
3. Alle neuen Sensoren haben `unique_id`?
