# ESP32 projekty: webové hodiny, monitor internetu a poznámky

Tři malé hobby projekty pro **Arduino Nano ESP32** (Arduino framework), které se hodí jako vstup do světa WiFi, webserverů a práce s časem na mikrokontrolérech.

> **Kompatibilita:** Všechny projekty vznikly a byly určeny pro desku **Arduino Nano ESP32**. Funkčnost na jiných deskách (např. jiné ESP32 vývojové desky) **nezaručuji**, i když je to pravděpodobně možné, případně s drobnými úpravami (např. piny RGB LED v projektu `internet_monitor`).

| Projekt | Co dělá |
|---|---|
| [`čas-Brno-web`](./čas-Brno-web) | Webová stránka s aktuálním časem (Brno / Europe/Prague), čas se bere z NTP |
| [`internet_monitor`](./internet_monitor) | Kontroluje WiFi a dostupnost internetu, stav ukazuje vestavěná RGB LED |
| [`poznamky-web`](./poznamky-web) | Webová aplikace na poznámky, které se ukládají do flash paměti ESP32 |

> **Poznámka o vzniku:** Kód v tomto repozitáři vznikl s asistencí AI. Pomáhal mi s ním Claude (od Anthropic). Návrh a úpravy jsem dělal ve spolupráci s ním, takže před použitím v jakémkoli důležitějším nasazení doporučuji kód projít a otestovat.

---

## Obsah

- [Požadavky](#požadavky)
- [Rychlý start](#rychlý-start)
- [Projekt 1: Webové hodiny (Brno)](#projekt-1-webové-hodiny-brno)
- [Projekt 2: Internet Connection Monitor](#projekt-2-internet-connection-monitor)
- [Projekt 3: Webové poznámky](#projekt-3-webové-poznámky)
- [Struktura repozitáře](#struktura-repozitáře)
- [Známá omezení](#známá-omezení)
- [Licence](#licence)

---

## Požadavky

- Deska **Arduino Nano ESP32** (na jiných deskách bez záruky, viz poznámka o kompatibilitě výše)
- [Arduino IDE](https://www.arduino.cc/en/software) (nebo `arduino-cli`)
- Podpora desek ESP32 nainstalovaná přes Boards Manager
- WiFi síť 2,4 GHz

Použité knihovny (`WiFi`, `WebServer`, `HTTPClient`, `Preferences`, `time.h`) jsou součástí ESP32 jádra pro Arduino, nic dalšího instalovat nemusíš.

## Rychlý start

1. Naklonuj repozitář:
   ```bash
   git clone https://github.com/jakub-obecny/arduino-projekty.git
   ```
2. Otevři příslušný `.ino` soubor v Arduino IDE.
3. Doplň údaje o své WiFi síti:
   ```cpp
   const char* ssid     = "VAŠE SSID";
   const char* password = "VAŠE HESLO";
   ```
4. Vyber správnou desku a port, nahraj sketch.
5. Otevři Serial Monitor (**115200 baud**).

> ⚠️ **Nikdy necommituj své skutečné WiFi heslo.** V repozitáři nech zástupné hodnoty.

---

## Projekt 1: Webové hodiny (Brno)

Soubor: `time-Brno-web/time-Brno-web.ino`

ESP32 se připojí k WiFi, synchronizuje čas přes NTP a spustí malý webserver na portu 80. Po otevření IP adresy desky v prohlížeči se zobrazí stránka s velkými zelenými číslicemi na tmavém pozadí, která si každou sekundu doptává aktuální čas.

### Jak to funguje

- **NTP:** čas se synchronizuje s `pool.ntp.org`.
- **Časové pásmo:** `CET-1CEST,M3.5.0/2,M10.5.0/3` (Europe/Prague), takže přechod na letní a zimní čas se řeší automaticky.
- **Endpointy:**
  - `GET /` vrací HTML stránku
  - `GET /time` vrací čas jako prostý text ve formátu `HH:MM:SS`
- Stránka pomocí JavaScriptu (`fetch`) volá `/time` každou sekundu.

### Použití

1. Nahraj sketch a v Serial Monitoru počkej na výpis `IP adresa: ...`.
2. Do prohlížeče (ve stejné síti) zadej `http://<IP adresa>/`.

Čas pro jiné město stejného pásma nebo jinou zónu změníš úpravou řetězce `timeZone`.

---

## Projekt 2: Internet Connection Monitor

Soubor: `internet_monitor/internet_monitor.ino`

Sketch pro **Arduino Nano ESP32**, který průběžně sleduje, jestli je deska připojená k WiFi a jestli skutečně funguje internet. Stav je vidět na vestavěné RGB LED, detaily jdou do Serial Monitoru.

### Význam LED

| Barva | Stav |
|---|---|
| 🟢 Zelená | Připojeno k WiFi i k internetu |
| 🔴 Červená | WiFi funguje, ale internet není dostupný |
| 🔵 Modrá (bliká) | Probíhá připojování k WiFi |
| ⚫ Zhasnuto | Není připojeno k WiFi |

### Jak to funguje

- Každých **10 sekund** (`checkInterval`) pošle HTTP GET na `testUrl` (výchozí `http://www.google.com`) s timeoutem 5 s.
- Pokud server odpoví jakýmkoli HTTP kódem, internet se považuje za dostupný.
- Při ztrátě WiFi se sketch automaticky pokouší znovu připojit.
- Do Serial Monitoru vypisuje stav WiFi, sílu signálu (RSSI) a výsledek testu.

### Nastavení

```cpp
const char* testUrl = "http://www.google.com"; // URL pro test připojení
const int checkInterval = 10000;               // interval kontroly v ms
```

### Poznámka k LED

Vestavěná RGB LED na Arduino Nano ESP32 má společnou anodu, proto kód hodnoty invertuje (`255 - hodnota`). Piny `LED_RED`, `LED_GREEN` a `LED_BLUE` jsou předdefinované v konfiguraci desky. Na jiné desce bude potřeba piny a logiku upravit.

---

## Projekt 3: Webové poznámky

Soubor: `poznamky-web/poznamky-web.ino`

ESP32 funguje jako malý poznámkový blok dostupný z prohlížeče v domácí síti. Poznámky se ukládají do flash paměti pomocí knihovny `Preferences`, takže **přežijí restart i odpojení napájení**. Každá poznámka si pamatuje datum a čas vytvoření (z NTP).

### Funkce

- přidání poznámky (max. **200 znaků**)
- zobrazení seznamu poznámek s datem a časem
- smazání jedné poznámky
- smazání všech poznámek (s potvrzením)
- limit **20 poznámek**
- responzivní stránka s fialovým přechodem na pozadí, použitelná i na mobilu

### Endpointy

| Cesta | Metoda | Popis |
|---|---|---|
| `/` | GET | HTML stránka aplikace |
| `/getNotes` | GET | Seznam poznámek jako JSON |
| `/addNote` | POST | Přidá poznámku (parametr `note`) |
| `/deleteNote?index=N` | GET | Smaže poznámku s indexem `N` |
| `/clearAll` | GET | Smaže všechny poznámky |

### Použití

1. Doplň WiFi údaje, nahraj sketch a v Serial Monitoru počkej na výpis IP adresy.
2. Otevři `http://<IP adresa>/` v prohlížeči.

---

## Struktura repozitáře

```
.
├── README.md
├── internet_monitor/
│   └── internet_monitor.ino
├── poznamky-web/
│   └── poznamky-web.ino
└── time-Brno-web/
    └── time-Brno-web.ino
```

Arduino IDE vyžaduje, aby se složka jmenovala stejně jako hlavní `.ino` soubor.

## Známá omezení

- WiFi údaje jsou zapsané přímo ve zdrojovém kódu.
- Webserver hodin nemá žádné zabezpečení ani HTTPS, je určený pro domácí síť.
- Monitor testuje dostupnost jen jedné URL. Pokud je právě ona nedostupná, bude se zobrazovat chyba internetu, i když zbytek světa funguje.
- Při neúspěšném připojení k WiFi hodiny a poznámky čekají donekonečna (bez timeoutu).
- **Poznámky:** aplikace nemá přihlášení, poznámky může číst i mazat kdokoli v síti. Mazání je řešeno přes GET požadavky a text poznámek se na stránce vkládá bez ošetření HTML, takže se hodí jen do důvěryhodné domácí sítě.
- **Poznámky:** časové pásmo je nastaveno pevným posunem (GMT+1 a +1 h letního času), takže se čas při přechodu na zimní čas sám nezmění. Projekt s hodinami používá řetězec `Europe/Prague` a tento problém nemá.
- Ukládání do flash paměti má omezený počet zápisů, pro časté zápisy to není ideální.


- Kód a dokumentace vznikly s pomocí AI asistenta **Claude** (Anthropic).
- ESP32 Arduino core od Espressif.
