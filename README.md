# TP Docker : audit de sécurité et durcissement de l'application Quiz Docker

**Auteur** : Mathis BOULOU, Master 1 DevOps, SUP de Vinci
**Application d'origine** : franklin-tutorials/docker-introduction (dossier `dockerfile-exercice2`)

## 1. Contexte et périmètre

L'application « Quiz Docker » est un site statique construit avec Vite et servi par nginx dans un conteneur. Elle fonctionne, mais son image et son conteneur n'avaient pas été conçus avec la sécurité en tête. Avant un déploiement en production, un audit a été réalisé, suivi d'un durcissement de la configuration, sans modifier le code de l'application.

**Périmètre audité** : le Dockerfile, l'image produite, la configuration du serveur web, les options d'exécution du conteneur et les dépendances JavaScript.

## 2. Méthode

1. Construction et lancement de la version de base (`Dockerfile.avant`).
2. Relevé de l'état initial : utilisateur du processus, droits d'écriture, en-têtes HTTP, restrictions d'exécution, taille de l'image, scan de vulnérabilités (Docker Scout), `npm audit`. Résultats bruts : [`docs/avant.txt`](docs/avant.txt).
3. Identification des risques et classement par gravité.
4. Correction (Dockerfile, `nginx.conf`, `.dockerignore`, options `docker run`).
5. Validation : l'application fonctionne toujours et les mesures prouvent l'amélioration. Résultats bruts : [`docs/apres.txt`](docs/apres.txt).

## 3. Constats de l'audit

| # | Constat | Risque | Gravité |
|---|---------|--------|---------|
| 1 | Image de build `node:18-alpine` : Node 18 n'est plus maintenu (fin de vie en avril 2025) | Failles connues non corrigées dans la chaîne de build | Élevée |
| 2 | Le processus maître nginx s'exécute en **root** | En cas de compromission, l'attaquant dispose de privilèges élevés ; évasion de conteneur facilitée | Élevée |
| 3 | Écoute sur le port **80** (port privilégié) | Impose des droits élevés au processus | Moyenne |
| 4 | Tags d'images **non épinglés** (`nginx:alpine`) | Build non reproductible ; une mise à jour de l'image peut introduire une faille ou casser l'appli | Moyenne |
| 5 | `npm install` au lieu de `npm ci` | Versions de dépendances pouvant varier d'un build à l'autre | Moyenne |
| 6 | Aucune configuration nginx : version du serveur visible, aucun en-tête de sécurité | Fuite d'information, exposition au clickjacking, au XSS et au MIME sniffing | Moyenne |
| 7 | Aucun `HEALTHCHECK` | Un conteneur bloqué n'est pas détecté | Faible |
| 8 | Exécution sans restrictions : système de fichiers inscriptible, capabilities par défaut, pas de limites CPU/RAM/processus | Persistance d'un attaquant sur disque, déni de service, élévation de privilèges | Élevée |
| 9 | Aucun scan de vulnérabilités de l'image | Contenu de l'image non maîtrisé | Élevée |
| 10 | Dépendances anciennes (`vite ^4.4.5`) | Failles possibles dans les outils de build (hors image finale grâce au multi-stage) | Moyenne |

**Point positif existant** : le build multi-étapes laisse Node, `npm` et les sources hors de l'image finale, ce qui réduit déjà la surface d'attaque.

## 4. Corrections appliquées

### Dockerfile
- `node:22-alpine` : version maintenue de Node (risque 1).
- `npm ci` : installation exacte du `package-lock.json`, build reproductible (risque 5).
- `nginxinc/nginx-unprivileged:stable-alpine` : image nginx officielle conçue pour fonctionner **sans root** (risques 2 et 3).
- Écoute sur le port **8080**, non privilégié (risque 3).
- `HEALTHCHECK` sur `/healthz` (risque 7).
- `USER nginx` explicite.

### nginx.conf
- `server_tokens off` : masque la version du serveur.
- `X-Content-Type-Options: nosniff` : empêche le MIME sniffing.
- `X-Frame-Options: DENY` : protège du clickjacking.
- `Referrer-Policy: no-referrer` : limite la fuite d'URL.
- `Content-Security-Policy` : restreint les sources de contenu pour limiter l'impact d'un XSS.
- Route `/healthz` pour la supervision (risque 6).

### .dockerignore
Exclusion de `node_modules`, `.git`, `.env`, des journaux et de la documentation : build plus rapide et aucun secret copié par erreur dans l'image.

### Options d'exécution
| Option | Rôle |
|--------|------|
| `-p 127.0.0.1:8089:8080` | Publication limitée à la machine locale (en production, un reverse proxy expose le service) |
| `--read-only --tmpfs /tmp` | Système de fichiers en lecture seule, seul `/tmp` est inscriptible (en mémoire) |
| `--cap-drop=ALL` | Suppression de toutes les capabilities Linux |
| `--security-opt no-new-privileges` | Interdit toute élévation de privilèges |
| `--memory=128m --cpus=0.5 --pids-limit=100` | Plafonne les ressources (anti déni de service, anti fork bomb) |
| `--restart unless-stopped` | Redémarrage automatique en cas d'arrêt inattendu |

## 5. Mise en œuvre

```bash
docker build -t quiz:secure .
docker run -d --name quiz-secu \
  -p 127.0.0.1:8089:8080 \
  --read-only --tmpfs /tmp \
  --cap-drop=ALL \
  --security-opt no-new-privileges \
  --memory=128m --cpus=0.5 --pids-limit=100 \
  --restart unless-stopped \
  quiz:secure
```
Application disponible sur http://localhost:8089.

## 6. Tests de validation

| Test | Commande | Résultat attendu |
|------|----------|------------------|
| L'application fonctionne | `curl.exe -s -o NUL -w "%{http_code}" http://localhost:8089` | `200` |
| Conteneur sain | `docker ps` | `(healthy)` |
| Processus non-root | `docker exec quiz-secu id` | `uid=101(nginx)` |
| Disque en lecture seule | `docker exec quiz-secu sh -c "touch /test"` | `Read-only file system` |
| Version masquée et en-têtes | `curl.exe -sI http://localhost:8089` | Pas de version nginx, 4 en-têtes de sécurité |
| Restrictions actives | `docker inspect quiz-secu` | `ReadonlyRootfs=true`, `CapDrop=[ALL]` |

Résultats réels enregistrés dans [`docs/apres.txt`](docs/apres.txt).

## 7. Comparatif avant / après

| Critère | Avant | Après |
|---------|-------|-------|
| Utilisateur du processus | root | nginx (uid 101) |
| Port d'écoute | 80 | 8080 |
| Version de Node (build) | 18 (fin de vie) | 22 (maintenue) |
| Installation des dépendances | `npm install` | `npm ci` |
| Système de fichiers | Inscriptible | Lecture seule (+ `/tmp` en mémoire) |
| Capabilities Linux | Par défaut | Aucune |
| Élévation de privilèges | Possible | Bloquée |
| Ressources | Illimitées | 128 Mo, 0,5 CPU, 100 processus |
| Version du serveur exposée | Oui | Non |
| En-têtes de sécurité HTTP | Aucun | 4 en-têtes |
| Surveillance de santé | Aucune | HEALTHCHECK |
| Exposition réseau | Toutes interfaces | 127.0.0.1 |
| Scan de vulnérabilités | Non réalisé | Docker Scout (voir `docs/`) |

La taille de l'image et le nombre de vulnérabilités avant/après sont consultables dans `docs/avant.txt` et `docs/apres.txt`.

## 8. Risques résiduels et recommandations

- **Noyau partagé** : un conteneur n'est pas une machine virtuelle. Maintenir l'hôte et le noyau à jour.
- **Pas de HTTPS** : en production, placer un reverse proxy ou un load balancer qui gère TLS (Traefik, Caddy, nginx).
- **Épinglage par digest** (`image@sha256:...`) pour garantir la reproductibilité, avec mise à jour automatisée (Renovate, Dependabot).
- **Scan automatique en CI/CD** (Trivy ou Docker Scout) avec échec du pipeline sur vulnérabilité critique.
- **Registry privé** et **signature des images** (Cosign).
- **Supervision et journalisation** centralisées, avec alertes.
- **Politique CSP** à ajuster si l'application charge un jour des ressources externes.
- En orchestration (Kubernetes), reprendre ces règles dans le `securityContext` et ajouter des NetworkPolicies.

## 9. Structure du dépôt
├── Dockerfile # version durcie
├── Dockerfile.avant # version de base (comparaison)
├── nginx.conf # configuration sécurisée
├── .dockerignore
├── docs/ # résultats bruts avant / après
└── (code de l'application)