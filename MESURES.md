# Séance 1 — Mesures et réponses (parties 2 à 6)
Matthieu Grenier · 2026-10-06

## Mesure 2 — Identité du compte Azure
| Élément | Valeur |
|---|---|
| Identifiant de l'abonnement | `dd5bb398-ec8a-4234-9cae-8a1f0a72fcd1` |
| Nom du locataire (tenant) | Ynov |

**Question 2.1** — Ma liste `az account list-locations` contient 109 entrées !

## Mesure 3 — Groupe de ressources
| Élément | Valeur |
|---|---|
| ProvisioningState | Succeeded |

**Question 2.2** — La création du groupe n'a pas consommé de crédit. Le cout est de 0 et j'ai toujours 88 dollars sur 88.

## Mesure 4 — Environnement Docker du poste

| Élément | Commande | Valeur |
|---|---|---|
| Version du client Docker | `docker version` | 27.5.1 |
| Version du démon (server) | `docker version` | 27.5.1 (Docker Desktop 4.38.0) |
| Architecture | `docker info` | x86_64 (amd64) |
| Images déjà présentes | `docker info` | 0 |
| Mémoire allouée au démon | `docker info` | 1.805 GiB |

Contexte : Docker Desktop 4.38.0 sur Windows, moteur Linux dans WSL2 (noyau 6.18.40.1-microsoft-standard-WSL2), 4 CPU.

## Partie 3
**Question 3.1** — En conteneur, `127.0.0.1` désigne l'interface loopback du conteneur lui-même : le trafic redirigé par `-p` arrive sur l'interface réseau du conteneur, donc seule une écoute sur `0.0.0.0` (toutes les interfaces) le reçoit.

**Question 3.1** - En conteneur, 127.0.0.1 désigne l'interface loopback du conteneur lui-même : le trafic redirigé par -p arrive sur l'interface réseau du conteneur, donc seule une écoute sur 0.0.0.0 (toutes les interfaces) le reçoit.

**Question 3.2** - Oui. EXPOSE est purement déclaratif (documentation + métadonnée ExposedPorts lue par docker run -P et par les outils) : c'est -p hôte:conteneur qui crée réellement la redirection de port. Sans EXPOSE, -p 8080:8000 fonctionne à l'identique ; on perd seulement l'information pour le lecteur et pour -P. Non vérifié expérimentalement (option facultative de l'énoncé) : réponse fondée sur la documentation Docker.

**Question 3.3** - Sans .dockerignore, docker build . envoie tout le dossier au démon comme contexte de build :

Durée : les 400 Mo du .venv/ sont transférés au démon à chaque build (étape « transferring context »), avant même la première instruction.
Image : avec COPY . ./ (Dockerfile naïf), le .venv/ est copié dans l'image : +400 Mo inutiles, un environnement compilé pour l'hôte (Windows) inutilisable sous Linux, et un risque de fuite de fichiers locaux. En plus, toute modification dans .venv/ invalide le cache de la couche COPY.

### Mesure 5 — Premier build
| Mesure | Valeur |
|---|---|
| Durée du premier build (cache vide) | 41,45 s |
| Taille de l'image | 192 MB |
| Nombre de couches (`docker history`) | 19 |
| Couche la plus lourde et son instruction | 85,2 MB |
| Taille de l'image de base seule | 185 MB |

**Question 3.4** — Image de base **185 MB** / image finale **192 MB** = **96,4 %**. Environ **7 MB** sont donc ajoutés par mon Dockerfile, dont **5,64 MB** pour `pip install`. Recoupement par `docker history` : le plus gros layer de base fait **85,2 MB**, tandis que la couche `pip install` ajoute **5,64 MB**. Conséquence : le levier principal pour réduire la taille (et la surface d’attaque) est le choix de l’image de base (slim, distroless, alpine), plutôt que l’optimisation de mes propres couches.


### Mesure 6 — Réponse /health (local)
| Mesure | Valeur |
|---|---|
| Code HTTP | 200 (`HTTP/1.1 200 OK`) |
| hostname | 63c6d78d2c38 |
| version | 0.1.0 |
| status | ok |

**Question 3.5** — hostname avant relance : `63c6d78d2c38` ; après `docker rm -f` + `docker run` : `84deb47fbd8c`. Il est identique au CONTAINER ID de `docker ps`.

**Question 3.6** — conteneur bis (port 8001, `-e APP_VERSION="0.2.0-test"`) : version `0.2.0-test`, hostname `1b8240c777cf` ; conteneur d'origine (port 8000) : version `0.1.0`. Aucune reconstruction.

### Mesure 7 — Cache de build
| Scénario | Durée | Étapes CACHED | pip install rejoué ? |
|---|---|---|---|
| A — bon ordre | 11,28 s | 3 | Non |
| B — mauvais ordre | 14,64 s | 1 | Oui |
| Écart A/B | +3,36 s (B plus lent, soit environ 30 %) | 2 étapes de moins |

**Question 3.7** — Docker réutilise une couche (étape `CACHED`) tant que l'instruction est identique à celle du build précédent et, pour `COPY`/`ADD`, que le contenu des fichiers copiés n'a pas changé (il compare leur empreinte). Dès qu'une couche est invalidée, elle est reconstruite, et toutes les couches qui la suivent sont rejouées, même si leurs propres instructions n'ont pas changé, car chaque couche dépend de celle d'avant. Dans mes mesures : avec le bon ordre, modifier `app.py` n'invalide que `COPY app.py` et ce qui suit (3 étapes CACHED, `pip install` non rejoué) ; avec `COPY . ./` placé avant l'installation, la même modification invalide cette couche et `pip install` est rejoué (1 seule étape CACHED).

**Question 3.8** — Mesure d'écart avec Flask : +3,36 s par itération (B plus lent que A), car `pip install` (7,5 s) est rejoué à chaque modification de code. Sur 20 itérations : 20 × 3,36 s ≈ 67 s, soit environ 1 minute, négligeable avec Flask.

### Mesure 8 — Utilisateur non-root
| Contrôle | Attendu | Observation |
|---|---|---|
| `id` dans le conteneur | uid non nul | uid=10001(appuser) gid=10001(appuser) groups=10001(appuser) |
| `.Config.User` | non vide | 10001 |
| Écriture dans /etc | refusée | touch: cannot touch '/etc/preuve_root': Permission denied |

**Question 3.9** — Si un attaquant obtient l'exécution de code dans mon conteneur, il hérite des droits du processus. En `appuser` (uid 10001), il est confiné : je l'ai vérifié, il ne peut pas écrire dans `/etc` (`Permission denied`), ni installer d'outils ou modifier la configuration du système du conteneur. En root, il pourrait tout cela, et si un volume est monté ou une capacité excessive accordée, il aurait de bien meilleures chances de s'échapper du conteneur vers l'hôte.

## Partie 4
### Mesure 9 — Registre ACR
| Élément | Valeur |
|---|---|
| loginServer | acrdermascanmg34.azurecr.io |
| SKU | Basic |
| État d'approvisionnement | Succeeded |
| Coût mensuel annoncé | à relever sur la page tarifaire ACR (valeur et date de consultation) |

**Question 4.1** —

| SKU | Stockage inclus | Fonctionnalités clés |
|---|---|---|
| Basic | 10 Go | Registre privé, push/pull, authentification Entra ID, webhooks ; débit et nombre de webhooks les plus bas |
| Standard | 100 Go | Mêmes fonctions que Basic, avec plus de débit et de webhooks |
| Premium | 500 Go | Tout Standard, plus géoréplication, Private Link (endpoints privés) et règles réseau, zones de disponibilité, chiffrement par clé gérée par le client, tokens à portée de dépôt, débit maximal |

Passage au Premium pour DermaScan : dès que le registre devra être privé, c'est-à-dire accessible uniquement depuis le réseau de la plateforme et non exposé à Internet (attendu pour un système manipulant des données de santé), ou dès que DermaScan déploiera dans plusieurs régions ou pays (géoréplication pour rapprocher les images des centres de dépistage et résister à la panne d'une région). Le volume seul ne le justifie pas : quelques images d'inférence tiennent largement dans les 10 Go du Basic.

**Question 4.2** — Après `az acr login`, mon `~/.docker/config.json` contient une entrée `auths` pour `acrdermascanmg34.azurecr.io` (vide, sans champ `auth`), la ligne `"credsStore": "desktop"` et `"currentContext": "desktop-linux"`. Le jeton d'accès ACR (valable environ 3 h, émis à partir de mon identité Entra ID) n'est donc pas dans le fichier : Docker Desktop le délègue au gestionnaire d'identifiants de Windows. Sans `credsStore`, il serait écrit dans le champ `auth` en base64, c'est-à-dire encodé mais non chiffré. Ce fichier reste à ne jamais committer : il révèle les registres auxquels j'ai accès, et dans une configuration sans `credsStore` il contiendrait un jeton réutilisable par n'importe qui, qui pourrait pousser ou tirer des images en mon nom pendant sa validité (par exemple remplacer une image par une version piégée). Les dépôts publics sont scannés en continu par des robots à la recherche de ce type de secret.

**Question 4.3** — Le compte administrateur est déconseillé en entreprise, même s'il est plus simple (un identifiant et un mot de passe suffisent) :

1. **Identité partagée et non traçable** : un seul couple identifiant/mot de passe donne accès à tout le registre. Il circule entre personnes et pipelines, donc on ne peut plus savoir qui a poussé ou supprimé une image. Pour retirer l'accès d'une seule personne, il faut changer le mot de passe pour tout le monde.

2. **Droits maximaux et secret longue durée** : ce compte a tous les droits (push, pull, suppression) sur tous les dépôts, sans moindre privilège possible, et son mot de passe n'expire pas. S'il fuite dans un script ou un fichier de configuration, il reste exploitable jusqu'à une rotation manuelle. Avec Entra ID (rôles `AcrPush` / `AcrPull`, identités managées), les accès sont nominatifs, limités au besoin, et fondés sur des jetons courts et révocables.

### Mesure 10 — Premier push
| Mesure | Valeur |
|---|---|
| Durée du push | 25,43 s (`Measure-Command`) |
| Couches poussées | 10 lignes `Pushed` (les lignes `Waiting` sont de l'affichage de progression) |
| IMAGE ID local = taggé ? | Oui, identiques : `940ffe89a90a` pour `dermascan-health:0.1.0` et `acrdermascanmg34.azurecr.io/dermascan-health:0.1.0`. `docker tag` ajoute un second nom au même contenu, sans copie ni reconstruction (192 MB inchangés) |
| Digest | sha256:940ffe89a90ab7f1947ca8ebd4f5dab23980b889905cbf497839220c835e0e86 (size: 856) |

**4.4 — Vérification** : 1 dépôt `dermascan-health`, 1 tag `0.1.0`, 3 manifestes :

| Digest | Type | Architecture / OS | Taille | Rôle |
|---|---|---|---|---|
| `sha256:940ffe89a90a…` | `oci.image.index.v1` | - | 0 | Index porté par le tag `0.1.0` (= digest du push) |
| `sha256:93c74ebfd252…` | `oci.image.manifest.v1` | amd64 / linux | 46 784 429 o | Image exécutable |
| `sha256:f8f94eaa0ea1…` | `oci.image.manifest.v1` | unknown / unknown | 1 009 o | Attestation de provenance BuildKit |

Aucun manifeste orphelin : les deux derniers sont référencés par l'index. Architecture amd64/linux lisible, donc pas de rebuild `--platform linux/amd64 --provenance=false` nécessaire (poste amd64).

### Mesure 11 — Exécution depuis ACR
| Mesure | Valeur |
|---|---|
| Durée du run avec téléchargement | 9,74 s |
| JSON identique à 3.4 ? | JSON de santé cohérent : status: ok, service: dermascan-health, version: 0.1.0 |
| hostname changé ? | Oui → hostname différent : f62bc655e9d5 |

**Question 5.1** - Le requirements.txt contient 1 paquet : Flask==3.0.3. Le pip freeze contient 7 paquets, soit 6 paquets supplémentaires : blinker==1.9.0, click==8.5.0, itsdangerous==2.2.0, Jinja2==3.1.6, MarkupSafe==3.0.4 et Werkzeug==3.1.9. Ces paquets proviennent des dépendances de Flask. L’audit des licences doit porter sur le pip freeze car il reflète les paquets réellement présents dans l’image, y compris les dépendances transitives, contrairement au requirements.txt qui ne contient ici que la dépendance directe.

**Question 5.2** - Flask 3.0.3 est sous licence BSD-3-Clause, CPython 3.12.7 sous licence Python Software Foundation (PSF), et les paquets système Debian utilisent différentes licences, notamment GPL, LGPL, BSD et Apache.

Les paquets système Debian présentent les obligations de licence les plus variées, notamment avec des composants sous GPL/LGPL. La redistribution doit donc respecter les obligations propres aux licences de chaque composant et conserver les notices correspondantes.

## Mesure 12 — Nettoyage
| Contrôle | Observation |
|---|---|
| Espace libéré par `docker system prune` | Non exécuté |
| Ressources actives dans $RG | 1 — acrdermascanmg34 (ACR), Succeeded |
| SKU et coût journalier | Basic — coût journalier non relevé |
| Date `a_detruire` | 2027-01-31 |
| Groupes de ressources au total | 1 |
| Crédit restant | 88 $ / 88 $ |
