# debian13-infra

Ansible-pull baseline en post-install configuratie voor **Debian 13 (Trixie)** desktops.

Dit is geen traditionele Ansible-repo met een inventory en push-model. In plaats daarvan gebruikt elke machine **`ansible-pull`**: de machine haalt zelf deze repository op en past de configuratie toe. Zo is elk systeem self-managing en altijd in sync met wat hier staat.

## 📋 Wat dit doet

Het `local.yml` bestaat uit vier plays:

| Play | Doel |
|---|---|
| **1. Basislijn** | Basispakketten, MOTD, Firefox policies (uBlock Origin + freeboot.nl), unattended-upgrades |
| **2. Hardware** | CPU-microcode, firmware, Wi-Fi/Bluetooth, GPU, audio, TLP, gebruikersgroepen |
| **3. Cinnamon** | Xorg, LightDM + slick-greeter, Cinnamon desktop, PipeWire, fonts, kantoor, AppArmor, UFW |
| **4. Dagelijks gebruik** | CUPS (printen), Blueman, VLC, KeePassXC, Flameshot, File Roller |

## 📂 Repository-structuur

```
debian13-infra/
├── local.yml                              # Hoofd-playbook (4 plays)
├── README.md
├── .gitignore
└── files/
    ├── motd.txt                           # Bericht van de dag
    ├── firefox/
    │   └── policies.json                  # uBlock Origin + freeboot.nl startpagina
    └── apt/
        └── 52unattended-upgrades-local    # Stille achtergrondupdates
```

## 🚀 Gebruik

### Handmatige run

Op een bestaande Debian 13-machine:

```bash
sudo apt install -y ansible git
sudo ansible-pull \
  --url https://github.com/henrydenhengst/debian13-infra.git \
  -C main \
  -i localhost, \
  -e desktop_user=henry \
  local.yml
```

- `-C main` — de branch die wordt uitgecheckt
- `-i localhost,` — impliciete inventory (let op de komma)
- `-e desktop_user=henry` — de gebruiker die in de hardware-groepen wordt gezet

### Tijdens een preseed-installatie

De preseed voert ansible-pull automatisch uit aan het einde van de Debian-installatie:

```bash
in-target sh -c 'DEBIAN_FRONTEND=noninteractive ansible-pull \
  --verbose --purge \
  --url https://github.com/henrydenhengst/debian13-infra.git \
  -C main -i localhost, -e desktop_user=freeboot \
  local.yml 2>&1 | tee /var/log/ansible-pull.log; exit ${PIPESTATUS[0]}'
```

Output komt terecht in `/var/log/ansible-pull.log` in het nieuwe systeem.

## 🔧 Vereisten

- Debian 13 (Trixie)
- `ansible` (bevat `ansible-pull`)
- `git`
- Collectie `community.general` (voor `debconf` en `ufw`):

```bash
ansible-galaxy collection install community.general
```

Het `ansible`-pakket uit Debian bevat deze collectie standaard. Alleen bij een `ansible-core`-only installatie moet je hem los installeren.

## 🔒 Secrets

Er horen **geen secrets** in deze repository.

Dit is een desktop-baseline, geen fleet-beheer. Geheimen zoals SSH-sleutels, WiFi-wachtwoorden, tokens of wachtwoorden worden lokaal gegenereerd of ingevoerd, niet via Ansible uitgerold.

De `.gitignore` blokkeert de meest voorkomende valkuilen (`*.vault_pass`, `.env`, `vault_pass.txt`, etc.).

## ✏️ Aanpassen

De belangrijkste variabelen die je mogelijk wilt wijzigen:

| Variabele | Waar | Default |
|---|---|---|
| `desktop_user` | Play 2, taak 9 | `freeboot` |
| Firefox startpagina | `files/firefox/policies.json` | `https://freeboot.nl` |
| MOTD-tekst | `files/motd.txt` | — |
| Update-gedrag | `files/apt/52unattended-upgrades-local` | auto-reboot uit |

Na een wijziging: committen, pushen, en de volgende `ansible-pull`-run pakt het automatisch op.

## 🔍 Verifiëren na een run

```bash
# Microcode (precies één van de twee)
dpkg -l | grep microcode

# Firmware
dpkg -l | grep firmware-misc

# AppArmor
sudo aa-status

# Firewall
sudo ufw status verbose

# Services
systemctl status lightdm cups apparmor

# Gebruikersgroepen
id freeboot

# Firefox policies (in Firefox)
about\:policies
```

## ⚠️ Bekende beperkingen

- **Schijfdetectie in de preseed**: De preseed detecteert dynamisch de eerste *niet-USB* schijf via `list-devices usb-partition` en `grep -v`. Dit werkt voor SATA (`/dev/sda`) en NVMe (`/dev/nvme0n1`). De USB-stick waar de installer vanaf boot wordt dus automatisch genegeerd. Uitzondering: een interne schijf in een USB-behuizing kan ten onrechte worden uitgesloten.
- **`ttf-mscorefonts-installer`** downloadt fonts tijdens de installatie. Zonder internet faalt deze taak.
- **Printen**: `cups-browsed` is bewust weggelaten. De moderne driverless printing-stack (`avahi-daemon` + `ipp-usb`) dekt de meeste netwerk- en USB-printers.
- **RAR**: `unrar` en `unrar-free` zijn niet opgenomen. Gebruik `7zip` voor RAR-bestanden (`7z x bestand.rar`), dat RAR5 ondersteunt.

## 🛠️ Werkinstructie: Debian 13 met preseed via netboot.xyz 3.0.3

## Doel

Een Debian 13 (Trixie)-installatie starten via netboot.xyz 3.0.3, waarbij de installer automatisch `preseed.cfg` ophaalt van `https://freeboot.nl/config/preseed.cfg`. De preseed voorziet in LVM/ext4-partitionering, installatie van Ansible en een volledige desktop-configuratie via `ansible-pull`.

## Vereisten

- Toegang tot netboot.xyz in UEFI-modus.
- Netwerktoegang vanuit de installer-omgeving tot freeboot.nl (DHCP of correcte netwerkinstellingen).
- Het preseed-bestand op `https://freeboot.nl/config/preseed.cfg` is bereikbaar en bevat de gewenste configuratie.

---

## Stap 1 — Start netboot.xyz op

Laat je machine booten van netboot.xyz in UEFI-modus en wacht tot het hoofdmenu verschijnt.

---

## Stap 2 — Open het Utilities-menu

Navigeer in het hoofdmenu naar **Utilities (UEFI)** en selecteer deze optie.

---

## Stap 3 — Stel de kernelparameters in

Kies **Kernel cmdline params**. Voer exact de volgende regel in en bevestig met Enter:

```
auto=true priority=critical url=https://freeboot.nl/config/preseed.cfg
```

**Toelichting (geverifieerd):**

- `url=` is een officiële alias voor `preseed/url=` en wordt door de Debian-installer herkend.
- `auto=true` en `priority=critical` zorgen voor een installatie met zo weinig mogelijk vragen, maar garanderen **geen volledige automatisering**. Of de installatie zonder tussenkomst verloopt, hangt af van de volledigheid van `preseed.cfg`.

**Bevestiging in de netboot.xyz-interface:**

Na het invoeren en bevestigen toont netboot.xyz de ingestelde parameters. Controleer of de regel er volledig en correct staat voordat je verder gaat.

---

## Stap 4 — Start de reguliere Debian 13-installatie

Ga terug naar het hoofdmenu en navigeer naar **Debian**. Kies de gewone Debian 13 (Trixie) installer (tekst- of grafische installatie). Kies **niet** de Live-omgeving; de Live-installer verwerkt preseed-bestanden niet op dezelfde manier.

---

## Stap 5 — Verifieer dat de kernelparameters zijn doorgegeven (verplicht)

Het instellen van parameters in het Utilities-menu garandeert niet automatisch dat de Debian-installer ze overneemt. De netboot.xyz-instelling geldt voor de daaropvolgende boot binnen dezelfde sessie, maar is niet gegarandeerd persistent na een herstart van netboot.xyz. Daarom is deze verificatie verplicht.

Voer de controle uit zodra de installer is gestart:

1. Open een shell in de installer (bijv. via het menu "Execute a shell" of met Alt+F2).
2. Draai:

   ```bash
   cat /proc/cmdline
   ```

3. Controleer of `url=https://freeboot.nl/config/preseed.cfg` (of `preseed/url=`) in de output staat.

### Als de parameter ontbreekt

- De Utilities-methode heeft de parameter niet doorgegeven aan deze boot.
- **Fallback 1:** Herhaal Stap 3 en controleer na het instellen expliciet of de parameter bewaard blijft bij het verlaten van het Utilities-menu.
- **Fallback 2:** Gebruik `preseed/url=` in plaats van `url=` (beide zijn geldig, maar `preseed/url=` is de canonieke vorm).
- **Fallback 3:** Als de Utilities-methode structureel niet werkt in jouw netboot.xyz-versie, is een custom iPXE-entry de betrouwbare route: definieer de kernelregel rechtstreeks in de netboot.xyz-configuratie of een eigen iPXE-script.

### Als de parameter aanwezig is maar de preseed niet wordt geladen

- Controleer de bereikbaarheid van `https://freeboot.nl/config/preseed.cfg` vanuit de installer.
- Controleer de systeemklok (HTTPS-certificaatvalidatie kan falen bij een foutieve klok).
- Voeg pas daarna tijdelijk `debian-installer/allow_unauthenticated_ssl=true` toe als troubleshooting-maatregel.

---

## ✅ Acceptatiecriteria (testchecklist)

| # | Criterium | Hoe te controleren |
|---|---|---|
| 1 | Preseed wordt opgehaald | Zichtbaar in de installer-log, of `cat /proc/cmdline` toont de parameter |
| 2 | LVM + ext4-partitionering | `lsblk`, `pvs`, `vgs`, `lvs` na de installatie |
| 3 | Ansible is geïnstalleerd | `ansible --version` |
| 4 | ansible-pull is uitgevoerd | `cat /var/log/ansible-pull.log` (laatste regels) |
| 5 | Gebruiker `freeboot` bestaat en zit in hardware-groepen | `id freeboot` |
| 6 | Geen onverwachte vragen | Installatie doorloopt zonder tussenkomst, afgezien van vragen die niet in de preseed zijn afgedekt |

---

## 💡 Aandachtspunten

- `priority=critical` is geen garantie voor volledige automatisering. Ontbreken essentiële waarden in `preseed.cfg`, dan stopt de installer alsnog met een vraag.
- De netboot.xyz Utilities-instelling is niet gegarandeerd persistent. De methode voegt de parameter toe aan de kernel command line van de volgende boot, maar of dit een herstart van netboot.xyz overleeft, is niet gedocumenteerd. Behandel het als een sessie-instelling en verifieer altijd met `cat /proc/cmdline`.
- HTTPS-fouten eerst onderzoeken, niet omzeilen. `debian-installer/allow_unauthenticated_ssl=true` is een workaround, geen oplossing. Controleer eerst de systeemklok, de certificaatketen en de bereikbaarheid van de preseed-URL.
- Deze procedure is een **testprocedure, geen bewezen end-to-end proces**. De stappen zijn gebaseerd op de gedocumenteerde werking van de Debian-installer en netboot.xyz, maar zijn niet end-to-end getest met netboot.xyz 3.0.3 in combinatie met jouw specifieke preseed. Voer de acceptatiecriteria daarom daadwerkelijk uit voordat je spreekt van een bewezen, volledig automatische installatie.

---

## 📌 Conclusie

Deze werkinstructie is een gedegen en realistische testprocedure. De twee grootste risico's zijn nu expliciet benoemd: de persistentie van de netboot.xyz-instelling en de volledigheid van de preseed. De verificatie via `cat /proc/cmdline` is verplicht gesteld, zodat je niet achteraf ontdekt dat de parameters nooit zijn doorgegeven. Pas na een geslaagde test — inclusief de controle dat de preseed daadwerkelijk wordt toegepast — kun je spreken van een bewezen, volledig automatische installatieprocedure.