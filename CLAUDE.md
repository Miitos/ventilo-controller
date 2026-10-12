# Ventilateur CREATE Windcalm — pilotage RF

Passerelle ESP32 + CC1101 qui rejoue les trames 433 MHz de la télécommande
d'un ventilateur de plafond, exposées en HTTP sur le LAN.

**État : fonctionnel.** Les 10 commandes de la télécommande sont capturées,
rejouées et validées sur le matériel.

---

## Matériel

| Élément | Détail |
|---|---|
| Ventilateur | CREATE WINDCALM M blanc — ASIN `B083WNZ4J7`, réf. `241319_771239` |
| | Version **sans Wi-Fi**. Aucun module Tuya, aucune API constructeur. |
| MCU | ESP32 DevKit V1 30 broches (ELEGOO, USB-C, puce série CP2102) |
| Radio | CC1101 433 MHz (E07-M1101D), antenne SMA |
| MAC | `5C:01:3B:BE:FE:10` |
| Hostname | `ventilo.local` |

### Câblage

| CC1101 (broche) | ESP32 (sérigraphie) | GPIO |
|---|---|---|
| 1 · GND | `GND` | — |
| 2 · VCC | `3V3` ⚠️ jamais 5V | — |
| 3 · GDO0 | `D4` | 4 — émission |
| 4 · CSN | `D5` | 5 |
| 5 · SCK | `D18` | 18 |
| 6 · MOSI | `D23` | 23 |
| 7 · MISO | `D19` | 19 |
| 8 · GDO2 | `D13` | 13 — réception |

GPIO13 plutôt que GPIO16 (sérigraphié `RX2`, prêtant à confusion) et plutôt
que GPIO12 (broche de configuration au boot, influence la tension flash).

L'émission ne peut passer **que** par GDO0 — contrainte de la puce CC1101.

---

## Protocole RF

Reverse-engineeré par capture, puis validé par rejeu.

- **433,92 MHz**, modulation **ASK/OOK**
- Codage PWM : impulsion ~740 µs = un bit, ~250 µs = l'autre. ~1 ms par bit.
- **32 bits** par trame : adresse (20) | touche (5) | compteur (3) | contrôle (4).
  Le compteur avance d'un cran à chaque appui (toutes touches confondues) ;
  le contrôle vaut `compteur XOR constante(touche)`. Le ventilateur ne vérifie
  pas le compteur → une trame capturée se rejoue indéfiniment.
- La télécommande répète la trame ~5 fois tant que le bouton est maintenu
- Séparateur de ~8 ms entre répétitions
- Les 24 premiers bits sont l'adresse du ventilateur, communs à toutes les
  commandes ; seuls les derniers bits identifient la touche
- Première répétition d'une pression : précédée d'une impulsion d'en-tête
  (~820 µs). **Ne pas utiliser cette trame-là** pour l'émission — prendre une
  répétition intermédiaire. C'est ce qui faisait échouer `tx_v1` initialement.

Émission : 8 répétitions, 8 ms d'intervalle. Valeurs validées empiriquement.

---

## Configuration

Un seul fichier : `ventilo.yaml`, plus `secrets.yaml` (Wi-Fi, hors versioning).

Structure :
- `cc1101` + `spi` — la radio
- `remote_receiver` sur GPIO13, `dump: raw` — laissé actif pour recapturer
- `remote_transmitter` sur GPIO4 — les blocs `on_transmit` / `on_complete`
  appelant `cc1101.begin_tx` / `begin_rx` sont **obligatoires** : ils
  basculent la puce entre modes. Sans eux, rien ne sort de l'antenne.
- 10 `script:` nommés `tx_*`, chacun portant une trame brute
- 10 `button:` template, un par script

### Commandes disponibles

`tx_power`, `tx_v1` à `tx_v6`, `tx_lum`, `tx_couleur`, `tx_sens`

---

## API HTTP

Base : `http://ventilo.local`

Format ESPHome : `/<domain>/<entity_name>/<action>` — le nom de l'entité est
celui du YAML **exactement**, espaces et majuscules compris.

```
POST /button/Power/press
POST /button/Vitesse%201/press   … jusqu'à Vitesse%206
POST /button/Lumiere/press
POST /button/Couleur%20lumiere/press
POST /button/Sens%20rotation/press
```

Réponse `200 OK`, corps vide. Encoder les noms côté client (`encodeURIComponent`,
`urllib.parse.quote`) plutôt que figer les `%20`.

`GET /events` — flux SSE (`ping`, `log`, `state`). Inutile ici (les boutons sont
sans état) mais c'est la brique pour une future synchro d'état.

**Contrôle d'Origin** : les requêtes navigateur cross-origin vers les endpoints
de contrôle sont rejetées. Les requêtes sans en-tête `Origin` (curl, backend,
app native) passent toujours. Une app web tierce doit donc relayer par un
backend.

Aucune authentification. Option `auth:` disponible sur `web_server`.

---

## Sémantique des commandes

| Commande | Nature |
|---|---|
| `tx_v1` … `tx_v6` | **Absolu.** Allume le ventilateur s'il est éteint et fixe la vitesse. |
| `tx_power` | **Toggle.** Bascule l'état. Validé : premier appui allume, deuxième éteint. |
| `tx_lum`, `tx_couleur`, `tx_sens` | Toggle / cyclique. |

Conséquence : pour un `turn_on` fiable, envoyer une vitesse, pas `power`.
`turn_off` n'a pas d'équivalent absolu.

Communication **unidirectionnelle** : aucun retour d'état du ventilateur.
Utiliser des commandes absolues, jamais de toggle côté app.

### Synchro d'état (firmware)

Le firmware tient l'état (entités template `Etat marche`, `Etat vitesse`,
`Etat lumiere`, `Etat couleur`, `Etat sens inverse`, restaurées au boot) :

- chaque bouton API appelle `apply_cmd` (paramètre `cmd` : 0 Power, 1-6
  vitesses, 7 lumière, 8 couleur, 9 sens) en plus de son script `tx_*` ;
- `remote_receiver.on_raw` décode les trames de la télécommande physique
  (classement par longueur d'impulsion > 500 µs = 1, fenêtre glissante de
  32 bits pour tolérer l'en-tête) et appelle `apply_cmd`. Comparaison aux codes
  ci-dessous **sans le compteur** : `code & 0xFFFFFF80` (adresse + touche) et
  `(code ^ code >> 4) & 0xF` (contrôle). Comparer les 32 bits exacts ne marche
  qu'une fois sur 8. Répétitions d'un appui (même code exact, < 400 ms)
  ignorées ; toute réception < 500 ms après une émission est ignorée (écho).

| Commande | Code 32 bits | | Commande | Code |
|---|---|---|---|---|
| Power | `0x21D18508` | | V4 | `0x21D18379` |
| V1 | `0x21D18269` | | V5 | `0x21D18481` |
| V2 | `0x21D18850` | | V6 | `0x21D18ABC` |
| V3 | `0x21D1866D` | | Lumière | `0x21D18962` |
| Couleur | `0x21D18E56` | | Sens | `0x21D188FA` |

Adresse commune : `0x21D18` (20 bits). Chaque trame compte 33 impulsions :
32 bits + une impulsion de fin courte.

---

## Pièges rencontrés

**ESPHome refuse Python ≥ 3.14.** Versions supportées : 3.11 à 3.13.
L'environnement actuel est un venv Python 3.13.

**`fan.template` n'a pas `on_turn_on` / `on_turn_off`.** Seuls `on_state` et
`on_speed_set` existent. Une config utilisant les premiers compile sans erreur
mais ne déclenche rien. L'entité `fan` a été abandonnée au profit de boutons.

**Bug ESPHome 2026.9.1 : `repeat` imbriqués avec `delay`.** Génère
`[](uint32_t iteration, uint32_t iteration)` — redéfinition, échec de
compilation. Contournement : aplatir le `repeat` interne.

**Capture : ne pas prendre la première trame d'une pression** (préambule).

---

## Workflow

```powershell
& $HOME\esphome-venv\Scripts\Activate.ps1
cd D:\Dev\ventilo-telecommande\ventilo-controller

esphome config .\ventilo.yaml   # valide sans compiler — toujours avant run
esphome run .\ventilo.yaml      # choisir l'option OTA
esphome logs .\ventilo.yaml
```

Premier flash en USB (COM3, pilote CP210x). Ensuite tout en OTA.

### Recapturer une trame

Les logs affichent `Received Raw:` sur deux lignes (`:026` puis `:035`) —
**les deux forment une seule trame**, il faut les concaténer.

Au boot, `CC1101 found! Chip ID: 0x0014` confirme la liaison SPI.
`FF0F`, `0000` ou `FFFF` = échec SPI, vérifier MISO/MOSI/CSN.

---

## Pistes

- Versionner le YAML (`secrets.yaml` dans `.gitignore`).
- Réservation DHCP pour une IP fixe.
- Automatisations horaires, intégration domotique.
