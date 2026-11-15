# hermes-veille-rss

Veille technologique et cyber **100 % locale** : des flux RSS/Atom résumés par un petit modèle hébergé localement via **Ollama**, planifiés par **[Hermes Agent](https://hermes-agent.nousresearch.com/)** et livrés sur **Telegram**.

```
📰 Veille — 18 nouveautés

- CISA ajoute une faille UnrealIRCd activement exploitée au catalogue KEV.
- Cursor confirme qu'une clé volée a permis de diffuser une extension trojanisée.
- …

🔗 CERT
• Multiples vulnérabilités dans les produits Citrix (13 novembre 2026)
https://www.cert.cyber.gouv.fr/avis/CERTFR-2026-AVI-1528/
…
```

## Principe

```
feeds.txt ──► veille.py ──► nouveaux articles (dédupliqués)
                  │
                  ├──► Ollama (LFM) : synthèse en français
                  └──► Telegram : synthèse + TOUTES les sources, en plusieurs messages si besoin
                  ▲
        Hermes cron (--no-agent) toutes les X heures
```

Choix de conception :

- **Le modèle ne fait que résumer.** La collecte, la déduplication et la liste des liens sont faites par le script : un petit modèle n'a pas besoin de *tool calling* et ne peut pas inventer de source.
- **Aucune dépendance** : Python 3, bibliothèque standard uniquement.
- **Silencieux sans nouveauté** : rien n'est envoyé si aucun nouvel article.
- **Pas de troncature** : Hermes tronque la sortie d'un script vers ~4000 caractères ; le script envoie donc lui-même le digest sur Telegram, découpé en messages sans jamais couper une source.

> ⚠️ La synthèse d'un petit modèle peut mélanger des articles ou ajouter des détails (numéros de CVE…). Les **sources** listées, elles, sont toujours exactes : vérifiez-y les faits.

## Prérequis

- Hermes Agent en Docker (image `nousresearch/hermes-agent`), gateway actif, Telegram configuré (`hermes setup`)
- Ollama sur l'hôte avec un modèle, par ex. `lfm3:latest`

## Installation

### 1. Ollama accessible depuis le conteneur

Ollama n'écoute que sur `127.0.0.1` par défaut. S'il tourne en service systemd :

```sh
sudo systemctl edit ollama
```
```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```
```sh
sudo systemctl restart ollama
```

Vérifier que le port 11434 n'est pas ouvert sur le LAN (`firewall-cmd --list-all`).

### 2. Hermes en Docker

Voir [`examples/docker-compose.yaml`](examples/docker-compose.yaml). Points clés :

| Réglage | Pourquoi |
|---|---|
| `PUID` / `PGID` | le conteneur tourne en UID 10000 par défaut ; aligner sur `id -u` / `id -g` |
| `:Z` sur le volume | SELinux (Fedora/RHEL) bloque sinon l'accès à `~/.hermes` |
| `extra_hosts: host.docker.internal:host-gateway` | joindre Ollama sur l'hôte |
| `TZ=Europe/Paris` | sinon les horaires cron sont en UTC |

Dans `~/.hermes/config.yaml`, pour utiliser Ollama comme modèle de Hermes :

```yaml
model:
  default: "{inserez-votre-model}"
  provider: "custom"
  base_url: "http://host.docker.internal:11434/v1"
cron:
  script_timeout_seconds: 900   # Ollama sur CPU peut être lent
```

### 3. Le script

```sh
mkdir -p ~/.hermes/scripts ~/.hermes/veille
cp scripts/veille.py ~/.hermes/scripts/ && chmod +x ~/.hermes/scripts/veille.py
cp examples/feeds.txt ~/.hermes/veille/feeds.txt
```

> SELinux : utiliser `cp` (pas `mv`), sinon le fichier garde un label que le conteneur ne peut pas lire.

Test manuel :

```sh
docker exec hermes python3 /opt/data/scripts/veille.py
```

### 4. Planification

```sh
docker exec hermes hermes cron create "0 8,13,18 * * *" \
  --no-agent --script veille.py --deliver telegram --name veille
```

## Configuration

### Flux — `~/.hermes/veille/feeds.txt`

```
# Thème | URL (RSS ou Atom)
CERT  | https://www.cert.ssi.gouv.fr/alerte/feed/
Cyber | https://www.bleepingcomputer.com/feed/
```

Un nouveau flux n'envoie que ses 3 derniers articles au premier passage.

### Variables d'environnement

| Variable                | Défaut                              | Rôle                                                                  |
| ----------------------- | ----------------------------------- | --------------------------------------------------------------------- |
| `VEILLE_MODEL`          | `lfm3:latest`                       | modèle Ollama                                                         |
| `VEILLE_OLLAMA_URL`     | `http://host.docker.internal:11434` | adresse d'Ollama                                                      |
| `VEILLE_NUM_CTX`        | `8192`                              | contexte du modèle ; à augmenter si beaucoup d'articles (ex. `32768`) |
| `VEILLE_MAX_ITEMS`      | `0` (tous)                          | nombre max d'articles par digest                                      |
| `VEILLE_SUMMARY_ITEMS`  | `0` (tous)                          | nombre d'articles passés au modèle pour la synthèse                   |
| `VEILLE_OLLAMA_TIMEOUT` | `600`                               | délai max de la synthèse (s) ; garder < `cron.script_timeout_seconds` |
| `VEILLE_SKIP`           | `canary\|nightly\|rc…`              | regex des titres ignorés (pré-releases)                               |
| `VEILLE_TELEGRAM_CHAT`  | `TELEGRAM_HOME_CHANNEL`             | chat ou channel de destination                                        |
| `VEILLE_DIRECT_SEND`    | `1`                                 | `0` = laisser Hermes livrer la sortie (tronquée à ~3500 car.)         |

Le token (`TELEGRAM_BOT_TOKEN`) et le chat sont lus dans `~/.hermes/.env`, déjà rempli par `hermes setup`.

## Utilisation

```sh
docker exec hermes hermes cron list            # jobs et prochaine exécution
docker exec hermes hermes cron run veille      # lancer maintenant (tick suivant, < 1 min)
docker exec hermes hermes cron remove veille   # supprimer
```

Forcer un digest complet (re-traiter tous les articles) :

```sh
mv ~/.hermes/veille/state.json ~/.hermes/veille/state.json.bak
docker exec hermes hermes cron run veille
```

## Dépannage

| Symptôme | Cause probable |
|---|---|
| `Status: silent (empty output)` | aucune nouveauté (normal) |
| `Permission denied: /opt/data` | `PUID`/`PGID` ou `:Z` manquant |
| `(synthèse indisponible : URLError)` | Ollama injoignable : `OLLAMA_HOST`, `extra_hosts`, pare-feu |
| `(synthèse indisponible : TimeoutError)` | modèle trop lent (CPU) : monter `VEILLE_OLLAMA_TIMEOUT` et `cron.script_timeout_seconds` (ou baisser `VEILLE_SUMMARY_ITEMS`) |
| `last_status: delivery_failed` | voir `last_delivery_error` dans `~/.hermes/cron/jobs.json` |
| `chat not found` / `Forbidden` | ID de chat erroné, ou bot non admin du channel |

Ne jamais mettre le **token** du bot dans `--deliver` : la cible attend un **ID de chat**.

## Fichiers

```
scripts/veille.py           script de veille
examples/feeds.txt          exemple de flux (CERT-FR, CISA, BleepingComputer…)
examples/docker-compose.yaml exemple de déploiement Hermes
```

État local (non versionné) : `~/.hermes/veille/state.json`.
