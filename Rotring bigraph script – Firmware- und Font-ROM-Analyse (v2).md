# Rotring bigraph script – Firmware- und Font-ROM-Analyse (v2)

Die Firmware RS1\_5.BIN (68008) wurde disassembliert, und ein Font-ROM (CS\_L1\_2.BIN) wurde mit der echten Firmware-Routine emuliert: Es passt. **Nichts davon wurde an der Hardware getestet.** Aussagen sind als *sicher* (im Code oder in der Emulation belegt) oder *vermutet* gekennzeichnet. Alle Adressen sind hexadezimal (0x…). Diese Fassung ersetzt die erste: Das DC-Format und der Befehl J sind korrigiert bzw. neu.

## 1. Überblick

- CPU 68008, ROM 64 KB (genutzt bis 0xC290), RAM 6264 (8 KB) ab 0x20000.
- C-Firmware (SCCS-Kennungen Mai bis November 1990, Version 4.x) plus Assembler-Soft-Float-Bibliothek.
- Kernstück ist ein **textbasierter Befehlsparser** (ein- und zweibuchstabige Befehle, Zeilenende CR). Schriften liegen im Font-Slot ab 0x30000 als Text in genau dieser Sprache.

## 2. ROM-Layout RS1\_5.BIN

| Bereich | Inhalt |
| --- | --- |
| 0x0000–0x00BF | Vektortabelle: SP = 0x21FFE, Reset-PC = 0x434; Level 2 → 0x762A; Level 6/7 → 0x75A2; alle übrigen → 0x761C (nur rte) |
| 0x00C0–0x03FF | 0xFF |
| 0x0390–0x058F | Zeittabelle (Wörter), Index = Wert aus 0x74000 (genutzt ab Index 0xD7) |
| 0x0400–0x0433 | Startcode (SP, A5, Variablen löschen, main bei 0x21F8) |
| 0x0434–0x0459 | Reset-Einstieg: 0x7C004 ← 1, 2704 Byte von 0xB800 nach 0x20000 kopieren, Sprung nach 0x400 |
| 0x0460 | Zeichenmaske klein: nur Steuercodes |
| 0x04A0 | Zeichenmaske voll: Codes 0–6, 8, 10–12, 14 sowie 0x20–0xFE |
| 0x04E0 | Mikroschritt-Tabelle: 40 Wörter, je zwei Spulenbytes (Vorzeichenbit + Sinus/Cosinus-Betrag, max. 0x69) |
| 0x1240–0x13B8 | Tastatur-Codetabellen |
| 0x13BB–0x21D6 | SCCS-Kennungen und eingebaute Befehlstexte (Rahmen-/Testzeichnung, NUL-getrennt) |
| 0x1920 | Initialstring "SC1 CR E CR" (beim Start geparst) |
| 0x21F8–0xA7FC | Programmcode |
| 0x7D33–0x8149 | Soft-Float-Routinen |
| 0x8132–0x8680 | Zustandstabelle des Parsers, 24 Byte pro Eintrag: Name(4), Handler, Folgezustände, Fehlerhandler |
| 0x868A–0x8940 | Befehlsnamen-Tabelle, 16 Byte pro Eintrag |
| 0xA7FC–0xB7FF | 0xFF |
| 0xB800–0xC28F | Initialwerte fürs RAM (0x20000–0x20A8F) |
| 0xC290–0xFFFF | 0xFF |

Module laut Kennungen (Auswahl): bvtreiber, glocon, f\_init, f\_ramtes, f\_romtes, f\_hauini, f\_pieper, f\_alpzei, f\_zeichen, f\_tabset/f\_tabfah, f\_tolaus/mit/obn/unt, f\_unters, f\_kreis, f\_gerade, f\_matcod, f\_tasber; bvetab, bveman, bvparser, bvhilfen, bvgeo.

## 3. Adressraum und Peripherie

| Adresse | Zugriff | Funktion | Sicherheit |
| --- | --- | --- | --- |
| 0x00000–0x0FFFF | R | ROM | sicher |
| 0x20000–0x21FFF | R/W | RAM; 0x20000–0x20A8F aus ROM initialisiert; Stack ab 0x21FFE abwärts | sicher |
| 0x20034–0x20133 | R/W | 256-Byte-Ringpuffer für Tasteneingabe | sicher |
| 0x202CC… | R/W | Schrittpuffer, 6-Byte-Einträge: X-Wort, Y-Wort, Stellbyte | sicher |
| 0x30000–0x3FFFF | R | Font-Slot, volles 64-KB-Fenster (belegt durch CS\_L1\_2) | sicher |
| 0x50000 / 0x50002 | W (Wort) | Motor X (Phasenwert aus Tabelle 0x4E0) | sicher (Funktion plausibel) |
| 0x60000 / 0x60002 | W (Wort) | Motor Y, analog | sicher |
| 0x70000 | R/W | Tastaturmatrix 8x8: Spalte 0x80…0x01 schreiben, Zeilen lesen, Code = Spalte\*8+Zeile | sicher |
| 0x74000 | W | Stellwert (Rampe 0xD7–0xF1, Ruhe 0xE0), zugleich Index der Zeittabelle | Funktion vermutet |
| 0x78000 | W | 1 in jedem Timer-Tick | vermutet (Watchdog/Quittung) |
| 0x7C000 | W | Hubmagnet, erste Stufe (0 = hoch, 1 = ab) | vermutet |
| 0x7C000 | R | 3 Bits invertiert (Konfiguration/DIP) | vermutet |
| 0x7C001 | W | Hubmagnet, zweite Stufe nach Wartezeit | vermutet |
| 0x7C002 | W | Beeper (Toggle in Zeitschleife) | sicher |
| 0x7C004 | W | Start: 1; Ende der Level-6/7-Routine: 0 | Funktion unbekannt |
| 0x7C005 / 0x7C006 | W | Ausgänge, zustandsabhängig | Funktion unbekannt |
| 0x7C007 | W | 1 in jedem Timer-Tick | vermutet |

Nicht zugeordnet: der Hallsensor für die Nullposition (vermutlich Lesebit an 0x70000 oder 0x7C000, oder Auslöser des Level-6/7-Interrupts).

## 4. Interrupts

- **Level 2 (0x762A):** Timer-Tick. Holt Schrittwerte aus dem Ringpuffer, schreibt 0x50000/0x60000 und 0x74000, quittiert über 0x78000 und 0x7C007 und ruft periodisch die Slot-/Font-Prüfung (0x7980) auf.
- **Level 6/7 (0x75A2):** setzt Flag Bit 4 in 0x2002C, hebt den Stift an, fährt Y-Schritte und setzt 0x7C004 auf 0. Auslöser vermutlich Anschlag oder Nullpunkt.

## 5. Befehlssprache

Zeilenorientiert, CR (0x0D) als Zeilenende. Zahlenformate (Code 0x54AA):

| Format | Bedeutung | Verwendung |
| --- | --- | --- |
| 0 | Rohtext bis CR, max. 32 Zeichen | NC |
| 1 | Hexadezimal | IK, IR, IS, IT, IU, DC |
| 2 | Dezimal mit Vorzeichen | IA, IC, IN |
| 3 | Festkomma, bis 4 Nachkommastellen, intern x10000 | Koordinaten X/Y |

Befehle laut Zustandstabelle: B, S, P, J, CC, CL, CR, DC, DM, D/, E, IA, IC, IK, IL, IN, IR, IS, IT, IU, IW, LA, LS, LW, MI, NC, PP, PR, PS, RP, RT, SC, SP, TA, TC, TH, TW, T/, W, F (ohne Wirkung).

Belegt durch eingebaute Texte und das Fremd-Font-ROM: `BX..Y..` fährt absolut, `SX..` und `SY..` (auch `SX..Y..`) bewegen relativ, `P3` setzt den Stift ab, `P0` hebt ihn, `CCX4` mit `CR`/`CL` zeichnet Kreise und Bögen, `E` beendet einen Block, `TH` setzt die Texthöhe, `TA` den Winkel, `J<hex>` ruft einen anderen Block auf (Offset relativ zum Imagestart).

## 6. Font-Slot-Format

### 6.1 Erkennung

Slot bei **0x30000**. Erstes Byte 0xFF bedeutet "kein Font": Standardwerte (Texthöhe 3,5, Winkel 90 Grad) und Fehler 0xF00B. Die ersten 16 Byte werden in 0x2022E zwischengespeichert, um einen Wechsel des Moduls zu erkennen.

### 6.2 Header

Beginnt bei 0x30000, besteht aus Befehlszeilen und endet mit `E` CR. Die Parameter landen in einer Struktur ab 0x20240:

| Befehl | Ziel | Bedeutung |
| --- | --- | --- |
| `NC<Text>` | 0x20240 | Fontname, bis 32 Zeichen (beide Varianten akzeptiert: Name direkt oder, wie im Original, Zeile NC1 plus Namenszeile) |
| `DC<hex4>` | 0x20268 | n = höchster Zeichencode. Direkt danach (ohne CR) 12\*n+12 Binärbytes, dann CR. Im Original: 256 Zeichen x 12 Byte Punktmatrix. Die Firmware überspringt den Block |
| `DM<dez>` | 0x20270 | unbekannt (im Original 1) |
| `IA<dez>` | 0x20274 | Flag (im Original 0) |
| `IC<dez>` | 0x20278 | unbekannt (im Original 37) |
| `IN<dez>` | 0x2027C | Anzahl Glyphen ab Code 0x20 (im Original 192) |
| `IK<hex>` | 0x20280 | Offset einer 192-Byte-Tabelle, im Original die Identität 0x00–0xBF |
| `IR<hex>` | 0x20284 | alternative Indextabelle, genutzt wenn Bit 1 in 0x2002C gesetzt |
| `IS<hex>` | 0x20288 | Prüfsumme (im Original nicht vorhanden) |
| `IT<hex>` | 0x2028C | Offset der Standard-Indextabelle |
| `IU<hex>` | 0x20290 | unbekannt |
| `RT<…>` | 0x20298 | Wert/10000 |
| `SC<…>` | 0x2029C | Skalierung, Standard 1,0 |
| `TA<…>` | 0x202A0 | Winkel, Standard 90 |
| `TC<…>` | 0x202A4 | Standard 0 |
| `TH<…>` | 0x202A8 | Höhe, Standard 0,35 (Anzeige x10 = 3,5) |
| `TW<…>` | 0x202AC | Breite, Standard 1,0 |
| `F<…>` | – | Füllzeile ohne Wirkung (im Original sieben Zeilen FFFFFF) |
| `E` | 0x20264 | Ende; Startzeiger der Daten wird hier abgelegt |

Nach dem `E` wird die Prüfsummen-Routine (0x5874) aufgerufen. **Ihr Ergebnis wird verworfen** (sie liefert immer 0): Die Prüfsumme wird nicht erzwungen.

### 6.3 Indextabelle und Glyphen

Adresse der Glyphe für Zeichencode c:

```text
G = 0x30000 + BE16( 0x30000 + IT + 2*(c - 0x20) )
```

Der Eintrag ist ein 16-Bit-Wert Big-Endian und ein Offset relativ zu 0x30000. Codes unter 0x20 oder mit c - 0x20 >= IN werden mit Fehler 0xF008 abgewiesen, nicht erlaubte Codes mit 0xF00A. Die Glyphe ist ab G eine Folge von Befehlszeilen (CR-terminiert) und endet mit `E` CR. Mit `J<hex>` kann sie andere Blöcke einbinden.

### 6.4 Größe

Das Original belegt das volle 64-KB-Fenster (Offsets bis 0x9CE2); ein 27512 passt. Der Prüfsummenbereich der Firmware reicht bis 0x3330F.

### 6.5 Validierung mit dem Original-ROM CS\_L1\_2.BIN

Das Fremd-ROM (64 KB, Name "001.DIN17/T") wurde mit der RS1\_5-Firmware emuliert (Unicorn, 68000):

- Der Header wird von der Routine 0x2176 mit Status 0 geparst. Aufbau: NC1, Namenszeile, IA00, IN192, IK0CA0, IT0D60, IR0EE0, sieben F-Zeilen, IC37, TH03.5, TW1.00, TA090, DM1, DC00FF mit 3072 Byte, CR, E.
- IT und IR zeigen auf je 192 Einträge zu 2 Byte; zwischen DC-Block und IT liegt die IK-Tabelle (192 Byte).
- Alle 247 verschiedenen Glyphblöcke beider Tabellen parsen fehlerfrei. Die Zeichensuche (0x2636) findet 0x20 bis 0xDF; 0xE0 bis 0xFE liefern 0xF008, 0xFF liefert 0xF00A.
- Beim Glyphenlauf liest die Firmware den DC-Block nicht.
- **Nicht geprüft:** das tatsächliche Zeichnen, also Maßstab, Einheiten und Zeichenbreite.

## 7. Offene Punkte

- Einheiten und Skalierung der Glyphkoordinaten (vermutlich mm, skaliert mit TH/SC).
- Bedeutung von IA, IC, IK, IU, DM, RT, TC und W; der DC-Block wird von dieser Firmware nicht benutzt.
- Hallsensor und endgültige Zuordnung der Ports 0x7C004–0x7C006.
- Ob die Hardware den Slot mit 27512 (64 KB) oder kleineren EPROMs befüllt: Adresslogik prüfen.

## 8. Werkzeuge (Download)

- **fontgen.py:** erzeugt aus einer Textdatei ein Font-ROM-Image (Header, Indextabelle, Glyphen, optional DC-Block); `--verify` bildet die Glyphensuche der Firmware nach.
- **fontcheck\_emu.py:** lässt die echte RS1\_5-Firmware ein Font-ROM parsen und testet alle Glyphen (Voraussetzung: Python-Paket unicorn).
- **beispiel\_font.txt:** Eingabebeispiel für den Generator (Platzhalter-Glyphen A und I, Geometrie ungetestet).
- **testfont.bin, testfont\_dc.bin:** daraus erzeugte 16-KB-Images, ohne und mit DC-Block; beide werden von der Firmware akzeptiert (Emulation).

Aufruf: `python3 fontgen.py beispiel_font.txt out.bin --name MEINFONT --dc 255 --size 65536` und danach `python3 fontcheck_emu.py RS1_5.BIN out.bin`.

## 9. Empfehlung für ein eigenes Font-ROM

1. Den Header des Original-ROMs als Vorlage nehmen (inklusive F-Zeilen und DC-Block), nur Name und Glyphen ersetzen.
2. Mit dem Generator und der Emulation testen, dann in ein 27512 brennen (EPROM-Adresse 0 = 0x30000).
3. Am Gerät beobachten: 0xF00B (Slot nicht erkannt), 0xF008 (Zeichen nicht vorhanden), 0xF00A (Zeichen nicht erlaubt).
