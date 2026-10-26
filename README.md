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

- `/dev/sda` in de preseed: werkt op SATA, niet op NVMe (`/dev/nvme0n1`). Laat `partman-auto/disk` en `grub-installer/bootdev` weg voor een universele preseed.
- `ttf-mscorefonts-installer` downloadt fonts tijdens de installatie. Zonder internet faalt deze taak.
- `cups-browsed` is verouderd; op moderne netwerken kan het printerdetectie verstoren. Verwijderen mag.
- `unrar-free` ondersteunt geen RAR5-archieven. Voor volledige support: gebruik `unrar` uit non-free.

## 📜 Licentie

Persoonlijk project — geen licentie. Gebruik op eigen risico.