# API — Ventilateur CREATE Windcalm

Passerelle ESP32 + CC1101 exposant les commandes de la télécommande en HTTP.

**Base URL** : `http://ventilo.local`
**Authentification** : aucune
**Protocole** : HTTP/1.1 — pas de TLS

---

## Format général

```
POST /button/<nom>/press
```

`<nom>` est le nom de l'entité **tel qu'il est défini dans la configuration**,
majuscules et espaces compris. Les espaces doivent être encodés (`%20`).

| | |
|---|---|
| Méthode | `POST` uniquement — un `GET` renvoie une erreur |
| Corps de requête | aucun |
| Paramètres | aucun |
| Réponse | `200 OK`, corps vide |

Chaque appel émet la trame radio 8 fois, à 8 ms d'intervalle. La commande
aboutit en ~300 ms.

---

## Endpoints

### Alimentation

| Commande | Endpoint |
|---|---|
| Power | `POST /button/Power/press` |

⚠️ **Bascule l'état.** Appelé sur un ventilateur allumé, il l'éteint ; sur un
ventilateur éteint, il l'allume. Voir « Gestion de l'état » plus bas.

### Vitesses

| Commande | Endpoint |
|---|---|
| Vitesse 1 | `POST /button/Vitesse%201/press` |
| Vitesse 2 | `POST /button/Vitesse%202/press` |
| Vitesse 3 | `POST /button/Vitesse%203/press` |
| Vitesse 4 | `POST /button/Vitesse%204/press` |
| Vitesse 5 | `POST /button/Vitesse%205/press` |
| Vitesse 6 | `POST /button/Vitesse%206/press` |

**Commandes absolues.** Elles fixent la vitesse et allument le ventilateur s'il
était éteint, quel que soit l'état de départ. Ce sont les seules commandes
idempotentes de l'API.

### Éclairage

| Commande | Endpoint | Effet |
|---|---|---|
| Lumière | `POST /button/Lumiere/press` | bascule allumé / éteint |
| Couleur | `POST /button/Couleur%20lumiere/press` | cycle les températures de couleur |

### Autres

| Commande | Endpoint | Effet |
|---|---|---|
| Sens de rotation | `POST /button/Sens%20rotation/press` | inverse le sens des pales |

---

## Gestion de l'état

Le ventilateur ne répond jamais : l'ESP32 émet, rien ne revient. Le firmware
**tient donc lui-même l'état**, à partir de deux sources :

- les commandes reçues par l'API (boutons ci-dessus) ;
- la **télécommande physique** : le `remote_receiver` décode chaque trame
  reçue, la compare aux 10 signatures connues et applique la commande
  (les répétitions d'un même appui sont ignorées).

L'état est conservé en flash (survit à un redémarrage).

| Entité | Lecture | Valeur |
|---|---|---|
| Marche | `GET /switch/Etat%20marche` | `value` : `true` / `false` |
| Vitesse | `GET /number/Etat%20vitesse` | `value` : `"1"` à `"6"` (chaîne) — dernière vitesse, même éteint |
| Lumière | `GET /switch/Etat%20lumiere` | `true` / `false` |
| Couleur | `GET /number/Etat%20couleur` | `"0"` chaude, `"1"` neutre, `"2"` froide (supposé) |
| Sens | `GET /switch/Etat%20sens%20inverse` | `true` = inversé |

```json
{"id":"switch/Etat marche","value":false,"state":"OFF"}
```

**Corriger l'état** sans rien émettre (ventilateur commandé hors de portée de
l'ESP32, ou état faux après un premier flash) :

```
POST /switch/Etat%20marche/turn_on        (ou turn_off)
POST /number/Etat%20vitesse/set?value=3
```

Le serveur exige un `Content-Length` sur les `POST` : `curl -X POST` sans
corps reçoit `411` (utiliser `-H "Content-Length: 0"`).

Limites restantes :

- **Portée** : un appui de télécommande que l'ESP32 n'entend pas n'est pas vu.
- **Toggles** : `Power`, `Lumiere`, `Sens` sont des bascules ; si l'état part
  faux, il reste faux jusqu'à correction.
- **Préférer les commandes absolues** pour allumer (une vitesse plutôt que `Power`).

---

## Flux d'événements

```
GET /events
```

Server-Sent Events. Trois types d'événements :

| Type | Contenu |
|---|---|
| `ping` | keep-alive |
| `log` | une ligne de log du firmware |
| `state` | JSON : `id` (`"switch/Etat marche"`), `name`, `value`, `state`, attributs |

À la connexion, le serveur envoie immédiatement l'état de toutes les entités,
puis chaque changement (télécommande physique comprise). C'est ce flux qu'écoute
l'application Windows (dépôt `ventilo-widget`).

---

## Exemples

### curl

```bash
curl -X POST "http://ventilo.local/button/Vitesse%203/press"
curl -X POST "http://ventilo.local/button/Lumiere/press"
```

Sous PowerShell, écrire `curl.exe` — `curl` est un alias vers
`Invoke-WebRequest`, dont la syntaxe diffère.

### Python

```python
import requests
from urllib.parse import quote

BASE = "http://ventilo.local"

def press(name: str) -> None:
    r = requests.post(f"{BASE}/button/{quote(name)}/press", timeout=3)
    r.raise_for_status()

press("Vitesse 3")
press("Lumiere")
```

### JavaScript

```javascript
const BASE = "http://ventilo.local";

async function press(name) {
  const res = await fetch(
    `${BASE}/button/${encodeURIComponent(name)}/press`,
    { method: "POST" }
  );
  if (!res.ok) throw new Error(`${res.status} ${res.statusText}`);
}

await press("Vitesse 3");
```

Encoder le nom plutôt que figer les `%20` : si une entité est renommée, seule
la constante change.

---

## Contraintes

### Origin — bloquant pour une application web

Le serveur applique un contrôle d'`Origin` sur les endpoints de contrôle et
d'état. Un `fetch()` depuis une page servie par un autre domaine est **rejeté**.
Le problème n'apparaît qu'au déploiement, pas en test local.

Les requêtes **sans en-tête `Origin` passent toujours** : curl, scripts,
backends, applications natives.

| Type de client | Fonctionne |
|---|---|
| Application native (mobile, desktop) | ✅ |
| Backend / script / tâche planifiée | ✅ |
| Page web servie par l'ESP32 (`js_include`) | ✅ même origine |
| Application web tierce | ❌ passer par un backend relais |

### Résolution du nom

`ventilo.local` passe par mDNS, qui n'est pas résolu partout — notamment sur
Android et dans certains runtimes serveur. Pour un client fiable, utiliser une
IP fixe (réservation DHCP sur la MAC `5C:01:3B:BE:FE:10`).

### Portée radio

L'ESP32 doit être à portée du ventilateur. Une commande hors de portée renvoie
quand même `200 OK` : la réponse confirme l'émission, pas la réception.

### Concurrence

Les scripts d'émission sont en mode `single` : un appel reçu pendant qu'une
émission est en cours est **ignoré silencieusement** (un avertissement
apparaît dans les logs). Espacer les appels d'au moins 400 ms.

### Sécurité

Aucune authentification. Quiconque atteint l'URL pilote le ventilateur.
Acceptable sur un LAN domestique ; à ne jamais exposer sur internet.
Le composant `web_server` accepte une option `auth:` si nécessaire.

---

## Codes de réponse

| Code | Signification |
|---|---|
| `200` | commande émise |
| `404` | nom d'entité inconnu — vérifier l'orthographe et l'encodage |
| `405` | méthode incorrecte — les actions exigent `POST` |

Un `200` garantit que la trame a été émise, jamais que le ventilateur l'a reçue.

---

## Évolutions possibles

**Noms sans espaces.** Renommer les entités (`Vitesse1`, `CouleurLumiere`)
supprimerait l'encodage des URLs. L'interface web resterait lisible.

**Authentification.** Option `auth:` sur `web_server` (basic, digest à partir de
2027.1.0 ; un client REST doit alors demander digest explicitement).