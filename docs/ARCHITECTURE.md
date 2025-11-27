# Systemarchitektur AutoTherm

## Übersicht

AutoTherm besteht aus drei Schichten:

```
┌─────────────────────────────────────────────────────────────┐
│                  Algorithmen (Julia)                        │
│                       OED · TiSR                            │
├─────────────────────────────────────────────────────────────┤
│                  Orchestrierung (Python)                    │
│           Zustandsautomat · Entscheidungslogik              │
├─────────────────────────────────────────────────────────────┤
│                     Hardware (Python)                       │
│     Densimeter · SoS-Sensor · Thermostat · Pumpe · Ventile  │
└─────────────────────────────────────────────────────────────┘
```

**Python** steuert die Anlage und koordiniert den Ablauf.  
**Julia** rechnet – Messplanung (OED) und Gleichungsentwicklung (TiSR).  
Die beiden kommunizieren über einen JSON-RPC Server.


## Regelkreis

Der Messablauf ist ein geschlossener Regelkreis:

```
    ┌──────────────────────────────────────────────────────────┐
    │                                                          │
    ▼                                                          │
┌───────┐    ┌───────┐    ┌────────┐    ┌───────┐              │
│  OED  │───▶│ Messe │───▶│Auswert.│───▶│ TiSR  │              │
└───────┘    └───────┘    └────────┘    └───┬───┘              │
    ▲                                       │                  │
    │                                       ▼                  │
    │                              ┌────────────────┐          │
    │                              │  Genau genug?  │          │
    │                              └───────┬────────┘          │
    │                                      │                   │
    │                           Nein ──────┴────── Ja          │
    │                             │                 │          │
    │                             ▼                 ▼          │
    └─────────────────── Plan anpassen          FERTIG         │
                                                               
```

1. **OED** berechnet optimale Messpunkte (T, p)
2. **Messen**: Anlage fährt Punkt an, misst Dichte und Schallgeschwindigkeit
3. **Auswertung**: Unsicherheitsrechnung, Daten speichern
4. **TiSR** fittet Zustandsgleichung an alle bisherigen Daten
5. **Prüfung**: Erfüllt die Gleichung die Genauigkeitsvorgaben?
   - Ja → Fertig
   - Nein → OED passt Plan an, weiter bei 2


## Zustandsautomat

Die Anlagensteuerung ist als endlicher Automat implementiert:

```
INIT
  │
  ▼
PLAN_OED ◀─────────────────────────────┐
  │                                    │
  ▼                                    │
SET_ISOTHERM ◀───────────────────┐     │
  │                              │     │
  ▼                              │     │
SET_PRESSURE ◀─────────┐         │     │
  │                    │         │     │
  ▼                    │         │     │
STABILIZE              │         │     │
  │                    │         │     │
  ▼                    │         │     │
MEASURE                │         │     │
  │                    │         │     │
  ▼                    │         │     │
EVALUATE ──────────────┘         │     │
  │        (nächster Druck)      │     │
  ▼                              │     │
FIT_EOS ─────────────────────────┘     │
  │        (nächste Isotherme)         │
  ▼                                    │
CHECK_QUALITY                          │
  │                                    │
  ├── Ja ──▶ COMPLETE                  │
  │                                    │
  └── Nein ──▶ UPDATE_OED ─────────────┘
```

Jeder Zustand hat eine klar definierte Aufgabe. Übergänge passieren nur bei Erfolg oder definierten Fehlern.


## Kommunikation Python ↔ Julia

Julia läuft als dauerhafter Server-Prozess. Python schickt Anfragen per HTTP/JSON:

```
Python                              Julia Server
   │                                      │
   │  POST /rpc                           │
   │  {"method": "oed.compute_plan",      │
   │   "params": {...}}                   │
   │ ────────────────────────────────────▶│
   │                                      │
   │  {"result": {"temperatures": [...],  │
   │              "pressures": {...}}}    │
   │ ◀────────────────────────────────────│
   │                                      │
```

Warum so und nicht direkt eingebettet?
- Julia-Startzeit ist lang (~3s), Server läuft durch
- Saubere Trennung, beide Seiten unabhängig testbar
- Kein PyJulia-Installationschaos


## Hardware-Anbindung

Alle Geräte werden über eine einheitliche Schnittstelle angesprochen:

```python
class Instrument(ABC):
    def connect(self) -> bool
    def disconnect(self)
    def read(self) -> MeasurementResult
    def get_status(self) -> dict
```

Konkrete Geräte erben davon und implementieren die Kommunikation (RS232, USB, Ethernet).

```
┌─────────────┐
│  Labor-PC   │
│   Ubuntu    │
└──────┬──────┘
       │
       ├── Ethernet/USB ──────▶ Anton Paar VTD (Dichte)
       │
       ├── USB ──────▶ SoS-Sensor (Schallgeschwindigkeit)
       │
       ├── RS232 ────▶ Thermostat
       │
       ├── RS485 ────▶ Drucktransmitter
       │
       └── USB-GPIO ─▶ Ventile, Pumpe
```


## Datenhaltung

```
data/
├── raw/              # Rohdaten direkt vom Gerät
├── processed/        # Ausgewertete Daten mit Unsicherheiten
└── results/          # Zustandsgleichungen, Berichte
```

- **SQLite** für Metadaten und schnelle Abfragen
- **HDF5** für große Rohdaten-Arrays (z.B. Oszilloskop-Traces)

Messdaten werden nicht ins Git-Repository eingecheckt.


## Konfiguration

Alle Parameter stehen in YAML-Dateien:

```yaml
# config/experiment.yaml
temperature:
  min_K: 253.15
  max_K: 533.15

pressure:
  min_MPa: 0.1
  max_MPa: 100.0

accuracy:
  eos_max_deviation: 0.002   # Ziel: 0.2%
```

Hardware-Adressen stehen in einer separaten Datei (`config/hardware.yaml`), die nicht ins Repo kommt.


## Offene Entscheidungen

Folgende Punkte sind noch zu klären:

- [ ] Konkretes Protokoll für Anton Paar VTD (Modell?)
- [ ] Format der OED-Ausgabe (von LUH übernehmen oder neu?)
- [ ] TiSR-Schnittstelle (welche Parameter, welches Output-Format?)
- [ ] Fehlerbehandlung bei Hardware-Ausfällen
- [ ] Logging-Strategie (Umfang, Rotation)


## Weiterführende Dokumente

- [Getting Started](GETTING_STARTED.md) – Entwicklungsumgebung einrichten
- [Hardware](HARDWARE.md) – Geräteliste und Protokolle
- [Coding Standards](CODING_STANDARDS.md) – Konventionen für Code
