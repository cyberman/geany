# Konzept: Portierung von Geany auf natives AmigaOS 3.2+ (ReAction statt GTK)

## 1. Zielbild und Rahmenbedingungen

Dieses Konzept beschreibt eine **native Portierung** von Geany auf AmigaOS 3.2+, mit folgenden harten Randbedingungen:

1. **Kein GTK** auf der Zielplattform.
2. Das UI wird vollständig mit **ReAction** umgesetzt.
3. Für den Editor-Kern (aktuell Scintilla) wird eine praktikable Ersatzstrategie benötigt.
4. Funktionsumfang darf bei Bedarf reduziert werden, solange ein stabiler, nativ bedienbarer Editor entsteht.

Nicht-Ziel in Phase 1 ist eine 1:1-Funktionsparität mit Geany auf modernen Desktop-Plattformen.

Zusätzliches Produktziel: Der Port soll sich für Amiga-Entwickler als **praktischer GoldEd-Ersatz**
etablieren (C/C++, ASM, Build-/Debug-Workflow), damit klassische Editoren mittelfristig in den
Ruhestand gehen können.

---

## 2. Technische Ausgangslage in Geany

Geany ist eng mit GTK verzahnt (Fenster, Menüs, Dialoge, TreeViews, Signals, Actions). Dazu kommt:

- **Scintilla** als Editor-Widget mit Lexer/Styling/Folding/Marker/Autocompletion.
- Build-/Run-/Projektlogik im Geany-Core.
- Plugin-Schnittstellen, die in Teilen UI-Annahmen mitbringen.

Daraus ergibt sich: Eine direkte "Recompile-and-run"-Portierung ist unrealistisch; erforderlich ist eine klare Entkopplung in Schichten.

---

## 3. Zielarchitektur (Portabilitätsschichten)

Empfohlen wird ein 3-Schichten-Modell:

### 3.1 Core-Schicht (plattformneutral)

Inhalt:

- Dokumentmodell (Datei laden/speichern, Encoding-Basis).
- Projektverwaltung (Basisfunktionen).
- Build-/Run-Konfiguration.
- Symbol-/Tag-Verwaltung (soweit ohne UI möglich).

Anforderung: Alle direkten GTK-Aufrufe aus dieser Schicht entfernen bzw. über Interfaces kapseln.

### 3.2 UI-Abstraktionsschicht

Neue, bewusst kleine API (z. B. `ui_backend.h`), die nur benötigt:

- Fenster/Tabs
- Menüs/Toolbar
- Dialoge (Open/Save, Optionen, Suche)
- Statusmeldungen
- Event-Dispatch

Implementierungen:

- `ui_backend_gtk` (bestehend, als Referenz)
- `ui_backend_reaction` (neu)

### 3.3 Editor-Engine-Schicht

Ein zweites Interface (z. B. `editor_backend.h`), das Editor-Funktionen kapselt:

- Textpuffer und Cursor/Selection
- Undo/Redo
- Styling (Syntaxfarben)
- Such-/Ersetzoperationen
- Marker/Bookmarks
- Optional: Folding, Margin-Symbole, Auto-Completion

Implementierungen:

- `editor_backend_scintilla` (bestehend)
- `editor_backend_amiga_native` oder `editor_backend_mini_scintilla` (neu)

---

## 4. GTK → ReAction Migrationsstrategie

### 4.1 UI-Mapping

| Geany/GTK-Konzept | ReAction-Äquivalent (Vorschlag) |
|---|---|
| MainWindow + Notebook | ReAction Window + ClickTab/PageGroup |
| Menüleiste | MenuClass/Intuition-Menüs |
| TreeViews (Datei/Symbole) | ListBrowser.gadget |
| Toolbar | SpeedBar/Buttons (plattformgerecht vereinfacht) |
| Statusbar | Layout + Text.gadget |
| Dialoge | ASL/FileRequester + ReAction-Dialogfenster |

Prinzip: Nicht GTK-Layout 1:1 nachbauen, sondern ReAction-konform vereinfachen.

### 4.2 Event-System

GTK-Signals durch Intuition/ReAction-Message-Loop ersetzen:

- zentraler Dispatcher pro Fenster
- Command-IDs für Aktionen (Datei, Bearbeiten, Suchen, Build)
- asynchrone/langlaufende Aktionen minimieren (AmigaOS 3.x typisch eher single-tasking-nah im UI)

### 4.3 Rendering-/Zeichenaspekte

- Fonts zuerst monospaced Bitmap/TT Engine, später optional erweiterbar.
- UTF-8 in Phase 1 nur "best effort" (fallback: ISO-8859-1/CP1252 + konfigurierbar).

---

## 5. Scintilla-Strategien (kritischer Pfad)

Es gibt drei realistische Optionen:

### Option A: Minimal-Scintilla-Port auf AmigaOS 3.2+

Idee:

- Scintilla-Backend ohne GTK/Qt/WinAPI, nur mit Amiga-Zeichen-/Event-Layer.
- Zielumfang: Cursor, Selection, Undo/Redo, Basis-Syntaxhighlighting.

Vorteile:

- Geany-Nähe bleibt hoch.
- Weniger invasive Änderungen im Core.

Risiken:

- Hoher Aufwand bei Rendering, IME/Keymap, Clipboard, Performance.
- Lexer-/Style-Komplexität auf schwacher Hardware.

### Option B: Ersatz-Editor-Engine (empfohlen für MVP)

Idee:

- Eigenes leichtgewichtiges Editor-Modul für ReAction/Intuition.
- Zunächst nur Kernfeatures.

Vorteile:

- Kontrolle über Ressourcenbedarf.
- Sauber auf Amiga-Bedürfnisse optimierbar.

Nachteile:

- Einige Geany-Features müssen entfallen oder neu implementiert werden.

### Option C: Hybrider Ansatz

- Kurzfristig: Ersatz-Engine (MVP)
- Mittelfristig: optionaler Scintilla-Port als austauschbares Backend

**Empfehlung:** Option C. Erst lauffähiges Produkt, danach Feature-Ausbau.

---

## 6. MVP-Funktionsumfang (bewusst reduziert)

### 6.1 Muss-Funktionen (Phase 1)

- Mehrere Dateien in Tabs
- Öffnen/Speichern/Neu
- Undo/Redo, Copy/Paste, Suchen/Ersetzen
- Zeilenzahlen
- Einfaches Syntaxhighlighting (z. B. C, C++, Python, Shell)
- Build-Befehl + Ausgabeprotokoll
- Werkzeugprofile für typische Amiga-Toolchains (z. B. vbcc, gcc, vasm, make)
- Projekt-Templates für Amiga-Programme (CLI-Tool, Workbench-App, Library-Skelett)

### 6.2 Soll-Funktionen (Phase 2)

- Projektverwaltung "light"
- Symbol-Liste auf Basis Tagmanager (ohne komplexe Live-Analyse)
- Konfigurierbare Tastaturkürzel
- Error-Parser für typische Compiler-Ausgaben (Sprung zur Fehlerzeile)
- Starten im Emulator/auf Zielsystem (z. B. UAE-Startkommando als konfigurierter Build-Schritt)

### 6.3 Kann entfallen (zunächst)

- Code-Folding
- Indikator-/Marker-Spezialfälle
- Erweiterte Auto-Completion/Calltips
- Vollständige Plugin-Kompatibilität
- VTE-integriertes Terminal

---

## 6a. Anforderungen, um GoldEd & Co realistisch abzulösen

Damit der Port im Alltag angenommen wird, reicht ein „technisch lauffähiger Editor“ nicht aus.
Folgende Punkte sind für einen Wechsel zentral:

1. **Schneller Build-Loop**
   - Ein-Klick- bzw. Ein-Shortcut-Build.
   - Fehlerausgaben parsebar und klickbar (Datei/Zeile).

2. **Amiga-typische Projektstruktur**
   - Vorkonfigurierte Include-/Lib-Pfade.
   - Toolchain-Profile pro Projekt (Debug/Release, CPU-Ziel, FPU-Flags).

3. **Stabile Bedienbarkeit auf klassischer Hardware**
   - Responsives Scrollen und Tippen auch bei begrenzter CPU.
   - Konservative Speicherpolitik (große Dateien mit Warnung).

4. **Niedrige Umstiegshürde**
   - Optionales „Classic-Keymap“-Preset (GoldEd-nahe Shortcuts).
   - Import von einfachen Projekt-/Build-Einstellungen aus bestehenden Workflows.

5. **Verlässliche Dateikodierung**
   - Klare Defaults (z. B. ISO-8859-1 oder UTF-8, je Projekt definierbar).
   - Sichtbare Anzeige der aktiven Kodierung im UI.

Messbarer Akzeptanzindikator: Ein bestehendes kleines bis mittleres Amiga-C-Projekt kann ohne
Editorwechsel vom Laden bis zum Build/Run durchgängig im Geany-Amiga-Port bearbeitet werden.

---

## 7. Plugins & Erweiterbarkeit

Empfehlung für Amiga-Port:

1. In Phase 1: **Plugins deaktivieren** oder nur statisch eingebaute "Core-Plugins" erlauben.
2. Neue Capability-Flags (z. B. `GEANY_CAP_EDITOR_MINIMAL`, `GEANY_CAP_NO_GTK`).
3. Später selektive Wiederaktivierung kompatibler Plugins.

Ziel: Stabilität vor Parität.

---

## 8. Toolchain-, Build- und Portierungsfragen

### 8.1 Buildsystem

- Parallelpflege von Meson/Autotools nicht für alle Targets nötig; Amiga-spezifische Buildprofile ergänzen.
- Klare `#ifdef`-Grenzen (`GEANY_TARGET_AMIGAOS`).

### 8.2 Abhängigkeiten minimieren

- glib-Funktionen möglichst beibehalten, wenn portabel verfügbar.
- GTK komplett ausschließen im Amiga-Target.
- Optionale Komponenten (VTE, manche Dialog-Subsysteme) im Amiga-Build deaktivieren.

### 8.3 Speicher-/Performancebudget

- Zielprofil explizit definieren (z. B. 68040/68060 + RTG).
- Große Dateien ggf. mit Warnschwelle.
- Syntaxhighlighting inkrementell und "on-demand".

---

## 9. Umsetzungsplan (Meilensteine)

### M0 – Architektur-Refactoring (4–8 Wochen)

- UI-Aufrufe im Core inventarisieren.
- `ui_backend`- und `editor_backend`-Interfaces einziehen.
- GTK-Implementierung auf neue Interfaces umhängen.

Ergebnis: Linux-Build weiterhin lauffähig, aber entkoppelte Architektur.

### M1 – ReAction-Shell (4–6 Wochen)

- Hauptfenster, Menü, Tabs, Dateioperationen.
- Basis-Dialoge und Event-Loop.

Ergebnis: "Viewer/Editor ohne Komfortfunktionen" auf AmigaOS.

### M2 – Editor-MVP (6–10 Wochen)

- Textbearbeitung, Undo/Redo, Suche/Ersetzen, Zeilenzahlen.
- rudimentäres Highlighting.

Ergebnis: nutzbarer Code-Editor.

### M3 – Geany-Kernintegration (6–12 Wochen)

- Build/Run, Projekt light, Logfenster.
- Tagmanager-Integration in reduzierter Form.
- Toolchain-Profile + Fehlerparser für Amiga-Compiler.

Ergebnis: Geany-ähnlicher Workflow.

### M4 – Stabilisierung & optionale Features (laufend)

- Performance-Tuning
- Optionaler Scintilla-Hybridversuch
- Plugin-Roadmap
- Feinschliff für „GoldEd-Ablösung“ (Keymap-Presets, Projektmigration)

---

## 10. Risiken und Gegenmaßnahmen

1. **Zu hohe Komplexität bei Scintilla-Port**  
   Gegenmaßnahme: MVP ohne vollständigen Scintilla-Funktionsumfang planen.

2. **UI-Entkopplung unterschätzt**  
   Gegenmaßnahme: Frühes Refactoring auf Host-Plattform testen, bevor Amiga-spezifische Arbeit beginnt.

3. **Ressourcenlimit der Zielhardware**  
   Gegenmaßnahme: Feature-Toggles, aggressive Vereinfachung, messbare Performance-Ziele.

4. **Wenig Manpower**  
   Gegenmaßnahme: Modulweise Arbeitspakete, klare "Definition of Done" je Meilenstein.

---

## 11. Entscheidungsvorlage (kurz)

Für eine realistische Umsetzung auf AmigaOS 3.2+ sollte das Projekt wie folgt entscheiden:

- **Ja** zu nativer ReAction-Oberfläche.
- **Ja** zu einer abstrahierten Editor-Schnittstelle.
- **Ja** zu einem reduzierten MVP-Feature-Set.
- **Nein** zu einer sofortigen 1:1-Replik aller GTK-/Scintilla-Funktionen.

Damit entsteht zuerst ein stabiler "Geany-Amiga-Edition" Kern, der anschließend iterativ erweitert werden kann.
