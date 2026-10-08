# Séance 1 — Réponses partie 1
Matthieu Grenier · M2 YNOV Montpellier · 2026-10-06

## Tableau de décisions

| Décision | Valeur |
|---|---|
| Suffixe personnel | `mg34` |
| Région Azure | `francecentral` (à confirmer en 2.2) |
| Groupe de ressources | `rg-dermascan-registre-mg34` |
| Registre ACR | `acrdermascanmg34` |
| Nom de l'image | `dermascan-health` |
| Tag initial | `0.1.0` |
| Image de base | `python:3.12-slim` |
| WORKDIR | `/app` |
| Port | `8000` |
| uid d'exécution | `10001` |

**Question 0.1** — Le nom du registre devient un sous-domaine DNS public (`<nom>.azurecr.io`) : un nom DNS est unique au niveau mondial, d'où l'unicité sur tout Azure et la restriction aux minuscules et chiffres.

## 1.1 — Modèles de service

| Modèle | Géré par le fournisseur | À ma charge | Exemple Azure |
|---|---|---|---|
| IaaS | Matériel, réseau physique, virtualisation | OS, mises à jour, runtime, application, données | Azure Virtual Machines |
| PaaS | Matériel, réseau, virtualisation, OS et correctifs, runtime | Code, configuration, données | Azure App Service |
| SaaS | Toute la pile, application comprise | Données, utilisateurs et droits d'accès | Microsoft 365 |

## 1.2 — Où s'arrête ma responsabilité

| Couche | Dernier modèle où c'est à moi |
|---|---|
| Correctifs de sécurité de l'OS | IaaS |
| Version du runtime Python | IaaS |
| Code de l'application | PaaS |
| Dimensionnement du nombre d'instances | PaaS |
| Données métier | SaaS |

**Question 1.1** — Les données métier restent à ma charge dans les trois modèles. DermaScan demeure responsable de traitement de données de santé (données sensibles, art. 9 RGPD) quel que soit le modèle : le fournisseur n'est que sous-traitant (art. 28), et la localisation UE, le chiffrement, la minimisation et la base légale restent de notre responsabilité.

## 1.3 — Caractéristiques NIST

| Caractéristique | Traduction opérationnelle |
|---|---|
| Libre-service à la demande | Je crée une ressource sans ouvrir de ticket ni attendre l'accord d'un humain. |
| Accès réseau large | J'administre et j'utilise le service depuis n'importe quel poste connecté, via portail, CLI ou API. |
| Mise en commun des ressources | Mon registre tourne sur une infrastructure partagée avec d'autres clients ; je choisis une région, pas une machine. |
| Élasticité rapide | J'augmente ou je réduis la capacité en quelques minutes, sans acheter de matériel. |
| Service mesuré | Chaque ressource est comptée et facturée tant qu'elle existe, et visible dans Cost Management. |

**Question 1.2** — Le service mesuré : tout ce qui existe est compté et facturé, même sans être utilisé. Une ressource oubliée continue de consommer le crédit, d'où le nettoyage obligatoire.

## 1.4 — Mesure 1 : exploration du portail

| # | Observation | Réponse |
|---|---|---|
| 1 | Champs obligatoires à la création d'un ACR | 6 : abonnement, groupe de ressources, nom du registre, emplacement, étendue d'étiquette du nom de domaine, plan de tarification |
| 2 | Identifiant et état de l'abonnement | `dd5bb398-ec8a-4234-9cae-8a1f0a72fcd1`, état `Enabled` |
| 3 | Nombre de groupes de ressources | 0 |
| 4 | Crédit restant au 2026-10-06 | 88 $ sur 88 $ (expiration le 06/10/2027) |
| 5 | Quota limitant pour 10 VM | _à relever_ (ex. « Total Regional vCPUs ») |
| 6 | Rôle de l'onglet Étiquettes | Associer des paires clé/valeur à une ressource pour l'imputer à un projet, un propriétaire ou une date de fin, et la retrouver ou filtrer les coûts. |

### Classement

| Service Azure | Modèle |
|---|---|
| Azure Virtual Machines | IaaS |
| Azure App Service | PaaS |
| Azure Container Registry | PaaS |
| Azure Machine Learning (espace managé) | PaaS |
| Microsoft 365 | SaaS |

**Question 1.3**
- Une ressource ne peut appartenir qu'à un seul groupe à la fois (elle peut être déplacée, pas partagée).
- Supprimer un groupe supprime toutes les ressources qu'il contient, de façon irréversible.
- Non : la région du groupe ne contraint pas celle de ses ressources. Elle fixe seulement où sont stockées les métadonnées du groupe (et donc où s'exécutent les opérations de gestion sur ce groupe).
