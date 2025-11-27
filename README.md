# AutoTherm

Forschungsplattform zur automatisierten Messung thermophysikalischer Stoffdaten mit simultaner Modellbildung.

## Worum geht es?

AutoTherm misst Dichte und Schallgeschwindigkeit von Fluiden über weite Druck- und Temperaturbereiche. Das Besondere: Die Anlage entscheidet selbst, welche Messpunkte als nächstes angefahren werden. Dafür laufen im Hintergrund zwei Algorithmen:

- **OED** (Optimal Experimental Design) wählt Messpunkte mit maximalem Informationsgehalt
- **TiSR** (Thermodynamics-informed Symbolic Regression) erstellt aus den Daten Zustandsgleichungen

Nach jeder Messreihe wird geprüft: Reicht die Genauigkeit? Falls ja, ist die Messung fertig. Falls nein, wird der Messplan angepasst und weitergemessen.

Erste Anwendung: Sustainable Aviation Fuels (SAF).

## Projektstruktur

```
autotherm/
├── config/           # Konfigurationsdateien (YAML)
├── docs/             # Dokumentation
├── src/
│   ├── python/       # Anlagensteuerung, Datenauswertung
│   └── julia/        # OED, TiSR
├── tests/
└── data/             # Messdaten
```

## Technologie

- **Python** für Hardwaresteuerung und Orchestrierung
- **Julia** für numerische Algorithmen (OED, TiSR)
- Kommunikation über JSON-RPC
- SQLite + HDF5 für Datenspeicherung

## Status

In Entwicklung – die Anlage wird im Rahmen eines BMFTR-geförderten Projekts aufgebaut.

## Dokumentation

- [Architektur](docs/ARCHITECTURE.md) – Systemaufbau und Datenfluss
- [Erste Schritte](docs/GETTING_STARTED.md) – Entwicklungsumgebung einrichten
- [Hardware](docs/HARDWARE.md) – Geräte und Schnittstellen

## Lizenz

Apache 2.0 – siehe [LICENSE](LICENSE).

## Kontakt

Prof. Dr.-Ing. Robin Wegge  
Technische Hochschule Georg Agricola  
Forschungsschwerpunkt Nachhaltigkeit in der Energietechnik
