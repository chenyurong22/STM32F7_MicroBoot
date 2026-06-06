# STM32F7 MicroBoot
 
Ein Git-Submodul mit den projektspezifischen Konfigurationsdateien und Hardware-Treibern für
die Integration des [OpenBLT](https://www.feaser.com/openblt/)-Bootloaders auf einem
**STM32F767ZI** (Nucleo-F767ZI Board). Der Firmware-Upload erfolgt über **CAN-Bus** mit dem
**XCP-Protokoll**, gesteuert über das Host-Tool **MicroBoot** von Feaser.
 
## Beschreibung
 
OpenBLT ist ein quelloffener Bootloader für Embedded-Systeme. Dieses Repository enthält
ausschließlich die targetspezifischen Anpassungen – die eigentliche OpenBLT-Bibliothek wird
separat als Submodul eingebunden. Die Konfiguration erfolgt zentral über `blt_conf.h`.
Alle Hardware-nahen Funktionen (CAN, Flash, Timer, LED, Watchdog) sind als Hook-Funktionen
implementiert, die OpenBLT an definierten Stellen aufruft.
 
## Dateien
 
| Datei              | Beschreibung                                                                 |
|--------------------|------------------------------------------------------------------------------|
| `blt_conf.h`       | Zentrale Bootloader-Konfiguration (CPU, CAN, Flash, Watchdog, Info-Tabelle)  |
| `blt_version.h`    | OpenBLT-Versionsdefinitionen                                                 |
| `boot.c/.h`        | Einstiegspunkt des Bootloaders, Initialisierung und Hauptschleife            |
| `app.c/.h`         | Anwendungsstart: Sprung in die User-Application                              |
| `can_t.c/.h`       | CAN-Treiber für STM32F7 HAL (CAN3, 500 kBit/s)                              |
| `flash.c/.h`       | Flash-Treiber: Löschen und Programmieren des internen STM32F7-Flashs         |
| `nvm.c/.h`         | NVM-Abstraktion über den Flash-Treiber                                       |
| `timer.c/.h`       | Systemtimer-Treiber (1 ms Zeitbasis)                                         |
| `cpu.c/.h`         | CPU-Treiber: Taktinitialisierung und Software-Reset                          |
| `com.c/.h`         | Kommunikationsschicht: Auswahl und Initialisierung des Transportkanals       |
| `cop.c/.h`         | Watchdog-Hook-Funktionen (IWDG)                                              |
| `xcp.c/.h`         | XCP-Protokollschicht: Seed/Key-Sicherheitsabfrage (deaktiviert)              |
| `backdoor.c/.h`    | Backdoor-Hook-Funktionen für erzwungenen Bootloader-Eintritt                 |
| `hooks.c`          | Implementierung aller Hook-Funktionen (Checksum, Infotable, COP, Backdoor)   |
| `led.c/.h`         | LED-Steuerung während des Bootloader-Betriebs                                |
| `infotable.c/.h`   | Info-Tabellen-Prüfung: Validierung der Ziel-Firmware vor dem Flashen         |
| `shared_params.c/.h` | Gemeinsame Parameter zwischen Bootloader und Application (RAM-basiert)     |
| `asserts.c/.h`     | Assert-Handler für Fehlerbehandlung                                          |
| `plausibility.h`   | Plausibilitätsprüfungen der Bootloader-Konfiguration zur Compile-Zeit        |
| `types.h`          | Grundlegende Typdefinitionen                                                 |
 
## Konfiguration (`blt_conf.h`)
 
### CPU
 
| Parameter                          | Wert      | Beschreibung                                      |
|------------------------------------|-----------|---------------------------------------------------|
| `BOOT_CPU_XTAL_SPEED_KHZ`          | 25000     | Quarzoszillator: 25 MHz                           |
| `BOOT_CPU_SYSTEM_SPEED_KHZ`        | 216000    | Systemtakt: 216 MHz                               |
| `BOOT_CPU_BYTE_ORDER_MOTOROLA`     | 0         | Little-Endian (Intel-Format)                      |
| `BOOT_CPU_USER_PROGRAM_START_HOOK` | 1         | Hook-Funktion vor Application-Start aktiviert     |
 
### CAN-Kommunikation
 
| Parameter                    | Wert    | Beschreibung                                          |
|------------------------------|---------|-------------------------------------------------------|
| `BOOT_COM_CAN_ENABLE`        | 1       | CAN-Transportschicht aktiviert                        |
| `BOOT_COM_CAN_BAUDRATE`      | 500000  | CAN-Baudrate: 500 kBit/s                              |
| `BOOT_COM_CAN_TX_MSG_ID`     | 0x7E1   | CAN-ID für Daten vom Steuergerät zum Host             |
| `BOOT_COM_CAN_RX_MSG_ID`     | 0x7E0   | CAN-ID für Daten vom Host zum Steuergerät             |
| `BOOT_COM_CAN_CHANNEL_INDEX` | 2       | CAN3 (zero-basiert: Index 2)                          |
 
### Flash und NVM
 
| Parameter                         | Wert   | Beschreibung                                            |
|-----------------------------------|--------|---------------------------------------------------------|
| `BOOT_NVM_SIZE_KB`                | 2048   | Gesamter Flash-Speicher: 2048 kB                        |
| `BOOT_NVM_CHECKSUM_HOOKS_ENABLE`  | 1      | Eigene Checksum-Überprüfung über Hook-Funktion          |
| `BOOT_FLASH_VECTOR_TABLE_CS_OFFSET` | 0x1F8 | Offset der Checksum-Adresse in der Vektortabelle       |
 
### Info-Tabelle
 
Die Info-Tabelle ermöglicht es, die Ziel-Firmware vor dem Flashen zu validieren. Die Tabelle
wird aus der Firmware-Datei ausgelesen und mit der aktuell im Flash gespeicherten Tabelle
verglichen. Über die Hook-Funktion `InfoTableCheckHook()` kann der Update-Vorgang bei
Inkompatibilität abgebrochen werden.
 
| Parameter               | Wert         | Beschreibung                                              |
|-------------------------|--------------|-----------------------------------------------------------|
| `BOOT_INFO_TABLE_ENABLE` | 1           | Info-Tabellen-Prüfung aktiviert                           |
| `BOOT_INFO_TABLE_LEN`   | 12           | Länge der Info-Tabelle: 12 Bytes                          |
| `BOOT_INFO_TABLE_ADDR`  | `0x08008200` | Adresse der Info-Tabelle in der User-Application          |
 
### Watchdog und Backdoor
 
| Parameter                   | Wert | Beschreibung                                                    |
|-----------------------------|------|-----------------------------------------------------------------|
| `BOOT_COP_HOOKS_ENABLE`     | 1    | Watchdog-Hook-Funktionen aktiviert                              |
| `BOOT_BACKDOOR_HOOKS_ENABLE`| 0    | Standard-Backdoor wird verwendet (kein eigener Hook)            |
| `BOOT_XCP_SEED_KEY_ENABLE`  | 0    | Seed/Key-Sicherheitsmechanismus deaktiviert                     |
 
## Shared Parameters
 
Über `shared_params` können Parameter aus dem RAM zwischen Bootloader und Application
ausgetauscht werden, ohne dass ein Flash-Schreibvorgang notwendig ist. Typische Verwendung:
Die Application setzt ein Flag im RAM, um den Bootloader beim nächsten Reset zum Verbleib im
Bootloader-Modus zu veranlassen.
 
## Info-Tabellen-Adresse und Vektortabelle
 
Die Adresse `0x08008200` für die Info-Tabelle ist **nicht** an eine bestimmte Flash-Aufteilung
gekoppelt. Sie folgt direkt auf den Interrupt-Vektor der User-Application: Der Interrupt-Vektor
des STM32F767 beginnt an der Startadresse der Application und belegt `0x200` Bytes (512 Byte,
da die Vektortabelle des Cortex-M7 größer ist als beim Standard-STM32F4). Die Info-Tabelle
liegt unmittelbar dahinter, also an `Startadresse + 0x200`.
 
Die Startadresse der Application selbst wird im übergeordneten Bootloader-Projekt festgelegt
und kann frei gewählt werden – `BOOT_INFO_TABLE_ADDR` muss dann entsprechend angepasst werden.

 
## Voraussetzungen
 
- [OpenBLT](https://github.com/feaser/openblt) – Bootloader-Kern (separat als Submodul einbinden)
- [MicroBoot](https://www.feaser.com/openblt/doku.php?id=manual:microboot) oder
[BootCommander](https://www.feaser.com/openblt/doku.php?id=manual:bootcommander) – Host-Tool
für den Firmware-Upload
- STM32CubeIDE
- CAN-Adapter mit XCP-Unterstützung (z. B. PCAN-USB)

## Abhängigkeiten
 
- `main.h` – STM32 HAL (inkl. CAN- und GPIO-Handle)

## Versionierung
 
Das Projekt führt zwei Versionsnummern in `blt_version.h`:
 
- **OpenBLT-Version**: die Original-Versionsnummer des OpenBLT-Parent-Projekts von Feaser,
die angibt, auf welchem Stand von OpenBLT dieses Submodul basiert.
- **Projektversion**: eine eigene Versionsnummer, die unabhängig davon gepflegt wird und
projektspezifische Änderungen, Erweiterungen und Anpassungen gegenüber dem OpenBLT-Original nachverfolgt.

## Quellen
 
Dieses Projekt basiert auf dem Demo-Projekt von:
- [Feaser/OpenBLT – STM32F7 Nucleo Demo](https://github.com/feaser/openblt)

## Lizenz
 
Dieses Projekt steht unter der [GPL-3.0 Lizenz](LICENSE).
