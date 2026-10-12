# ventilo-controller

Firmware ESPHome d'une passerelle **ESP32 + CC1101** qui pilote en 433 MHz un
ventilateur de plafond CREATE Windcalm (version sans Wi-Fi) et expose ses
commandes en HTTP sur le réseau local.

- Rejoue les 10 touches de la télécommande : Power, vitesses 1 à 6, lumière,
  couleur de la lumière, sens de rotation.
- **Synchro d'état** : décode aussi les appuis de la télécommande physique et
  tient l'état réel (marche, vitesse, lumière, couleur, sens), publié en direct
  sur `/events`.

Client Windows : [ventilo-widget](https://github.com/Miitos/ventilo-widget).

## Matériel

| Élément | Détail |
|---|---|
| MCU | ESP32 DevKit V1 (30 broches) |
| Radio | CC1101 433 MHz (E07-M1101D) + antenne SMA |

| CC1101 | ESP32 |
|---|---|
| GND | GND |
| VCC | 3V3 ⚠️ jamais 5 V |
| GDO0 | GPIO4 (émission) |
| CSN | GPIO5 |
| SCK | GPIO18 |
| MOSI | GPIO23 |
| MISO | GPIO19 |
| GDO2 | GPIO13 (réception) |

## Mise en route

ESPHome demande Python 3.11 à 3.13.

```powershell
python -m venv $HOME\esphome-venv
& $HOME\esphome-venv\Scripts\Activate.ps1
pip install esphome

Copy-Item secrets.example.yaml secrets.yaml   # puis renseigner le Wi-Fi
esphome config .\ventilo.yaml                 # validation
esphome run .\ventilo.yaml                    # 1er flash en USB, ensuite OTA
```

## Utilisation

```powershell
curl.exe -X POST "http://ventilo.local/button/Vitesse%203/press"
curl.exe "http://ventilo.local/switch/Etat%20marche"
```

Toutes les commandes, l'état et le flux d'événements : [API.md](API.md).
Protocole radio, câblage et pièges rencontrés : [CLAUDE.md](CLAUDE.md).

## Fichiers

| Fichier | Rôle |
|---|---|
| `ventilo.yaml` | Configuration ESPHome (radio, trames, boutons, synchro d'état) |
| `secrets.example.yaml` | Modèle de `secrets.yaml` (ignoré par git) |
| `API.md` | Documentation de l'API HTTP |
| `CLAUDE.md` | Notes techniques : protocole RF, câblage, pièges |

⚠️ L'API n'a pas d'authentification : ne jamais l'exposer hors du réseau local.
