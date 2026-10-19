# AI-Thinker ESP32-CAM: Performance-Testloop

## Ziel und Sicherheitsgrenzen

Die dedizierte AI-Thinker-Testkamera (klassischer ESP32, 4 MB Flash/PSRAM)
wird auf `vmmint` gebaut, seriell geflasht und gemessen. Die Produktivkamera,
Prusa-Connect-Zugangsdaten und andere USB-Geräte bleiben unberührt.

Auf `vmhost` laufen zwei CH340-Adapter an den physischen USB-Pfaden
`1-2.4.2.2` und `1-2.4.2.3`. Proxmox-VM 102 (`vmmint`) hat aktuell noch
keinen von ihnen durchgereicht. **Vor jeder USB-Konfiguration und jedem
Flash** muss der Testadapter durch passives serielles Firmware-Log oder andere
eindeutige Merkmale identifiziert werden. VID:PID `1a86:7523` genügt nicht.
Der einmalige Neustart von VM 102 ist für den Passthrough freigegeben.

Vor Experimenten: automatischen GPIO0/Reset-Bootloader-Einstieg prüfen,
vollständiges 4-MB-Flash-Backup und ein erfolgreich getestetes
Recovery-Verfahren außerhalb des Git-Repos sichern. Falls der Adapter keine
automatische Bootloader-Steuerung erlaubt, **keinen unbeaufsichtigten
Flash-Loop starten**. OTA ist kein Recovery-Ersatz; Bootloader-Rollback ist
nicht als aktiv nachgewiesen. `/upload` akzeptiert derzeit offenbar ohne
Basic-Auth-Prüfung Firmware; OTA erst nach Absicherung automatisieren.

## Reproduzierbarer Loop

Arbeitszweig: `integration/open-prs-ai-thinker`. Auf `vmmint` exakt diesen
Zweig auschecken und Arduino-ESP32-Core, Bibliotheken und Compiler-Versionen
festhalten. Testartefakte, Konfiguration mit Zugangsdaten und Flash-Backups
bleiben außerhalb des Repos. Nach Identitätsprüfung und Recovery-Probe:

1. Aktuellen Stand als Baseline bauen und flashen; Bootlog, Firmware-Version,
   Kamera-Modell, Sensor- und JPEG-Einstellungen dokumentieren.
2. Messpunkte in Firmware und einen Host-Testtreiber ergänzen: JPEG-Dekodierbarkeit,
   Bildaktualität, FPS und p95-Frame-Abstand, JPEG-Größe, Aufnahme-/Kopier-/
   TLS-/Upload-Zeit, RSSI, freien und größten zusammenhängenden internen
   Heap- und PSRAM-Block, Resets und Fehlerzähler.
3. Pro Kandidat genau eine Maßnahme ändern und je dreimal dieselben Szenarien
   ausführen: 10 Minuten Stream, fünf Minuten langsamer Client, 30 Minuten
   periodische Fotos. Zusätzlich SD an/aus und wiederholte Client-Abbrüche
   dort testen, wo die Änderung sie betrifft.
4. Bei Bootfehler, Reset-Schleife, ungültigem JPEG oder monotonem
   Speicherverlust abbrechen und das bekannte Image seriell zurückspielen.
   Nur reproduzierbare Verbesserungen ohne relevante Verschlechterung von
   Bildaktualität, p95-Frame-Abstand oder Speicherreserve übernehmen.
5. Jeden übernommenen Kandidaten eine Stunde, den kombinierten Stand 24
   Stunden prüfen. Für alle zehn Punkte Ergebnis und Entscheidung festhalten.

Der Testtreiber darf weder Token/Fingerprint noch rohe `/json_input`-Antworten
speichern: Diese Antwort enthält sensible Konfiguration. Messungen dürfen die
Verfügbarkeit der Kamera nicht durch unbeschränktes Polling verschlechtern.

## Zehn Maßnahmen, in Implementierungsreihenfolge

### 1. Frame-Besitz und Kopie

`CaptureStream()` alloziert und kopiert derzeit jeden JPEG-Frame vollständig
in PSRAM, obwohl periodische Cloud-Fotos während des Streams pausieren.
Diese Kopie im normalen Stream-Pfad entfernen. Einen eindeutigen Besitzer
und genau einen Rückgabepunkt für jeden `camera_fb_t` festlegen; Capture-,
Disconnect-, Destruktor- und Fehlerpfade inklusive Semaphore-Release testen.
Manuelle Cloud-Sendeaufträge während des Streams erst danach abarbeiten;
ein manueller Einzelbildaufruf während des Streams meldet „beschäftigt“.
Akzeptanz: keine Vollkopie pro Frame, keine Double-Returns, keine Leaks oder
permanent gehaltenen Semaphoren.

### 2. Stream-Callback

Feste `delay(1)`-Aufrufe und serielles Per-Frame-Logging entfernen. FPS nur
periodisch aus Zählern melden. Multipart-Header und JPEG-Längen bei Teil-
und langsamen Writes prüfen. Zunächst nur einen aktiven Stream-Client
zulassen. Falls Capture danach weiter als Blockierer messbar ist, einen
einzelnen Kamera-Producer mit höchstens einem wartenden Frame verwenden;
keine unbeschränkte Queue und kein Busy-Wait im Netzwerk-Callback.
Akzeptanz: HTTP bleibt während des Streams responsiv, p95-Frame-Abstand sinkt.

### 3. JPEG-Parameter

Die reale Laufzeitkonfiguration erfassen. QVGA als Werkswert unangetastet
lassen; QVGA/VGA/SVGA mit interner Sensorqualität 10/15/20 auf gleicher Szene
vergleichen. Größe, Aufnahmezeit, Stream und Upload gegen die Lesbarkeit der
Druckdetails abwägen. Nur eine klar bessere und visuell brauchbare Einstellung
empfehlen oder als Vorgabe übernehmen. Keine zusätzliche Software-JPEG-
Kodierung; der Sensor liefert bereits JPEG.

### 4. SD-Logging

Logzeilen über eine begrenzte Queue und niedrig priorisierten Writer bündeln.
`flush()` spätestens alle zwei Sekunden oder vor Neustart/Update statt nach
jeder Zeile; Karten- und Kapazitätsprüfung pro Block. Bei voller Queue
Verbose vor Info verwerfen, Fehler sichtbar erhalten und Drop-Zähler führen.
Keine rekursive Writer-Protokollierung. Mit fehlender/voller SD-Karte und
Schreibfehler testen. Der mögliche Verlust der letzten Logsekunden bei
Stromausfall wird dokumentiert.

### 5. WLAN

RSSI, JPEG-Größe, Sendezeit, Disconnects und Stream-Latenz korrelieren.
`WiFi.setSleep(false)` isoliert gegen Stromverbrauch, Temperatur und
Stabilität testen. Nur bei Nettogewinn übernehmen. Schlechte Funkabdeckung
als Infrastruktur-/Antennenproblem ausweisen, nicht mit CPU-Optimierungen
verwechseln. Physische Antennen- oder AP-Änderungen sind separat zu
autorisieren.

### 6. Doppelte Fotoaufnahme

Die vor jedem Cloud-Foto verworfene Trainingsaufnahme zeitlich messen und
probeweise entfernen. Frische/Bildinhalt nach Pause, Stream-Ende und
kontrolliertem Helligkeits- oder Blitzwechsel prüfen. Den Retry kurzer/kaputter
Frames so korrigieren, dass er tatsächlich neu aufnimmt und Puffer freigibt.
Bei alten oder schlecht belichteten Bildern Trainingsaufnahme beibehalten
oder auf nachgewiesene Sonderfälle begrenzen.

### 7. Cloud-HTTP/TLS

Handshake, Send und Antwortlesen getrennt instrumentieren. Teil-Schreibvorgänge
bis zur `Content-Length` vollständig senden, keine zusätzlichen Bytes nach
dem Body. Antwort mit Größen- und Zeitlimits gemäß HTTP-Framing lesen, statt
bis Verbindungsende zu warten. Keep-alive nur bei nachgewiesener Server-
Unterstützung und stabiler TLS-Speicherreserve aktivieren; nach jedem Fehler
neu verbinden. Der Geräte-Info-Upload ist nicht Teil jedes Fotozyklus.

### 8. Core-Belegung und Prioritäten

Laufzeit/Stack-High-Water-Mark der acht auf Core 0 gepinnten Tasks messen.
Foto-/Upload-Task und anschließend Telemetrie-/Serial-/LED-Tasks einzeln
auf Core 1 testen; hohe Prioritäten nur bei unverändertem Watchdog-Verhalten
senken. WLAN-Verwaltung zunächst unverändert lassen. Keine pauschale
Task-Migration; jeden Schritt gegen Stream und Cloud-Upload vergleichen.

### 9. Zwei Kamerapuffer

Erst nach 1 und 2 `fb_count=2` bei JPEG gegen `fb_count=1` testen. FPS und
p95-Frame-Abstand müssen gegenüber PSRAM-Reserve, größtem freien Block,
WLAN- und Langzeitstabilität gewinnen. Bei Speicher- oder Upload-Regression
auf einen Puffer zurückkehren. Kein PSRAM-DMA für klassischen ESP32 erwarten.

### 10. Compiler und Framework

Bestehende Arduino-Toolchain pinnen und Größe, RAM sowie Laufzeit erfassen.
Nur unterstützte Optimierungsflags oder gemessene Hotspots als einzelne
Varianten vergleichen. ESP-IDF-Portierung nur bei belegtem, relevantem
Mehrwert gegenüber Portierungs- und Kompatibilitätskosten beginnen.
Assembler/Sprachwechsel nur bei einem nachgewiesenen CPU-Hotspot.
AI-Thinker baut bereits mit 240 MHz CPU und 80 MHz Flash.

## Ergebnis

Pro Maßnahme: Git-Commit, Konfiguration, drei Wiederholungsläufe,
Messwertvergleich, visuelle JPEG-Prüfung und Entscheidung „übernommen“,
„ohne Gewinn verworfen“ oder „durch Hardware/Umgebung begrenzt“. Keine
Änderung an der Produktivkamera. Erst nach bestandenem Recovery- und
Langzeittest einen Firmwarestand als hardwarevalidiert bezeichnen.
