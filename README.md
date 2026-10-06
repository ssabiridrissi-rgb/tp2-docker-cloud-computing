# TP2 Docker / Cloud Computing

**Saad Sabir Idrissi**

Projet de conteneurisation, optimisation et déploiement d'un modèle de Machine Learning avec **Docker**, **Docker Compose** et **FastAPI**.

---

## 1. Objectifs du TP

Ce TP a pour objectifs de :

- conteneuriser un modèle de Machine Learning ;
- construire une image Docker pour l'entraînement ;
- construire une image Docker pour le service d'inférence ;
- analyser et optimiser la taille des images ;
- améliorer les temps de reconstruction grâce au cache Docker ;
- utiliser Docker Compose pour orchestrer les services ;
- exposer le modèle via une API FastAPI ;
- analyser les couches Docker avec Dive ;
- analyser les vulnérabilités avec Trivy ;
- comparer les différentes optimisations appliquées.

---

# 2. Architecture du projet

Le projet est composé de deux services principaux :

```text
                    Docker Compose
                          │
             ┌────────────┴────────────┐
             │                         │
             ▼                         ▼
      ┌──────────────┐          ┌──────────────┐
      │    train     │          │    serve     │
      │              │          │              │
      │ Entraînement │          │   FastAPI    │
      │ du modèle    │          │  Inférence   │
      └──────┬───────┘          └──────┬───────┘
             │                         │
             │       /artifacts        │
             └───────────┬─────────────┘
                         ▼
                  model.joblib
                  metrics.json
```

Le service `train` entraîne le modèle et génère les artefacts dans `/artifacts`.

Le service `serve` utilise ensuite le modèle généré afin de fournir les prédictions via une API FastAPI.

Docker Compose garantit que le service `serve` démarre après la réussite du service `train`.

---

# 3. Structure du projet

```text
tp2-docker-cloud-computing/
│
├── data/
│   └── ...
│
├── src/
│   ├── train.py
│   ├── predict.py
│   └── ...
│
├── Dockerfile.train
├── Dockerfile.serve
├── Dockerfile.serve.opt
├── docker-compose.yml
├── requirements.txt
├── benchmark.sh
├── .dockerignore
├── .gitignore
└── README.md
```

Le dossier `artifacts/` contient les fichiers générés pendant l'entraînement :

```text
artifacts/
├── model.joblib
└── metrics.json
```

Ces fichiers générés ne sont pas versionnés dans Git.

---

# 4. Images Docker

## Image d'entraînement

```text
adult-train:naif
```

Construite à partir de :

```text
Dockerfile.train
```

## Image de service naïve

```text
adult-serve:naif
```

Construite à partir de :

```text
Dockerfile.serve
```

## Image optimisée

```text
adult-serve:opt6
```

Construite à partir de :

```text
Dockerfile.serve.opt
```

L'image optimisée est également publiée sur GitHub Container Registry :

```text
ghcr.io/ssabiridrissi-rgb/adult-serve:opt6
```

---

# 5. Construction des images

Image d'entraînement :

```powershell
docker build -f Dockerfile.train -t adult-train:naif .
```

Image naïve :

```powershell
docker build -f Dockerfile.serve -t adult-serve:naif .
```

Image optimisée :

```powershell
docker build -f Dockerfile.serve.opt -t adult-serve:opt6 .
```

Pour afficher les images :

```powershell
docker images
```

---

# 6. Optimisation progressive de l'image Docker

Les optimisations ont été appliquées progressivement afin de mesurer leur impact sur :

- la taille de l'image ;
- le temps du cold build ;
- le temps du rebuild.

| # | Optimisation | Taille | Cold build | Rebuild |
|---|---|---:|---:|---:|
| 0 | Version naïve | 625.00 MB | 63.51 s | 63.73 s |
| 1 | `python:3.11-slim` | 261.26 MB | 63.65 s | 74.04 s |
| 2 | `COPY requirements.txt` avant `src/` | 261.26 MB | 106.41 s | 4.23 s |
| 3 | `.dockerignore` | 261.25 MB | 73.18 s | 2.92 s |
| 4 | `--no-cache-dir` | 169.84 MB | 89.81 s | 3.86 s |
| 5 | Multi-stage build | 165.76 MB | 80.00 s | 2.29 s |
| 6 | Non-root + HEALTHCHECK + ENTRYPOINT | 166.17 MB | 76.13 s | 3.79 s |

---

# 7. Réponses aux questions du TP

## Q1. Pourquoi le rebuild est-il presque aussi long que le cold build dans la version naïve ?

Dans la version naïve, le code du projet est copié avant l'installation des dépendances.

Lorsqu'un fichier du projet est modifié, Docker doit reconstruire les couches suivantes, notamment celle correspondant à l'installation des dépendances.

Les résultats obtenus sont :

| Version | Cold build | Rebuild |
|---|---:|---:|
| Version naïve | 63.51 s | 63.73 s |

Le rebuild est donc presque aussi long que le cold build.

Pour améliorer ce comportement, `requirements.txt` est copié avant le code source. Ainsi, les dépendances peuvent rester dans le cache Docker lorsque seul le code source est modifié.

---

## Q2. Quelle optimisation apporte le plus gros gain en taille d'image ? Quelle optimisation apporte le plus gros gain en temps de rebuild ? Pourquoi ces gains sont-ils différents ?

### Gain en taille

La plus grande réduction de taille est obtenue avec l'utilisation de :

```text
python:3.11-slim
```

L'image passe de :

```text
625.00 MB
```

à :

```text
261.26 MB
```

Cette optimisation apporte donc la réduction de taille la plus importante.

### Gain en temps de rebuild

Le plus gros gain sur le temps de rebuild vient de l'ordre des instructions dans le Dockerfile.

Avec la séparation entre `requirements.txt` et le code source, le rebuild passe à :

```text
4.23 s
```

puis les optimisations suivantes permettent de réduire davantage le temps, jusqu'à :

```text
2.29 s
```

avec le multi-stage build.

### Pourquoi les gains sont-ils différents ?

Les deux optimisations n'agissent pas sur le même aspect.

La taille de l'image dépend principalement des fichiers, de la base utilisée et des dépendances présentes dans les différentes couches.

Le temps de rebuild dépend surtout de l'utilisation du **cache Docker**.

Ainsi :

- `python:3.11-slim` réduit principalement la taille de l'image ;
- l'ordre `COPY requirements.txt` puis `COPY src/` permet principalement de conserver les dépendances dans le cache lorsque le code change.

---

# 8. Analyse avec Dive

L'image finale analysée avec Dive est :

```text
adult-serve:opt6
```

Les résultats obtenus sont :

| Élément | Résultat |
|---|---:|
| Efficiency Score | 98 % |
| Taille totale affichée par Dive | 517 MB |
| Espace potentiellement gaspillé | 11 MB |
| Plus grosse couche | 386 MB |

La plus grosse couche correspond à :

```text
COPY /install /usr/local
```

Elle représente environ :

```text
386 MB
```

Cette taille vient principalement des dépendances Python utilisées par le projet, notamment les bibliothèques scientifiques et les dépendances nécessaires à FastAPI.

Le score d'efficacité obtenu est de **98 %**, avec environ **11 MB d'espace potentiellement gaspillé**.

---

# 9. Résultats de l'entraînement

Le modèle a été entraîné dans le conteneur Docker.

| Métrique | Valeur |
|---|---:|
| Accuracy | 0.8773 |
| F1-score | 0.7263 |
| AUC | 0.9298 |
| Nombre d'échantillons d'entraînement | 26048 |
| Seed | 42 |
| Durée d'entraînement Docker | 0.53–0.64 s |

Ces résultats correspondent au modèle entraîné dans le cadre du TP.

Le modèle généré est enregistré dans :

```text
/artifacts/model.joblib
```

---

# 10. Test de l'API

L'API FastAPI est exposée sur le port :

```text
8000
```

Le test réalisé donne :

| Test | Résultat |
|---|---|
| `/health` | 200 OK |
| Prediction | `<=50K` |
| Probabilité | 0.0005 |
| Latence | 7.64 ms |
| Port | 8000 |

L'API peut être testée depuis la machine hôte avec :

```powershell
Invoke-RestMethod http://localhost:8000/health
```

L'endpoint de disponibilité du modèle peut également être vérifié avec :

```powershell
Invoke-RestMethod http://localhost:8000/ready
```

Une réponse de type :

```json
{
  "statut": "pret",
  "modele": "/artifacts/model.joblib",
  "duree_chargement_s": 0.76
}
```

indique que le modèle est chargé et que le service est prêt.

La documentation interactive FastAPI est disponible à :

```text
http://localhost:8000/docs
```

---

# 11. Docker Compose

Docker Compose permet d'orchestrer les services `train` et `serve`.

Le service `train` construit le modèle et écrit les artefacts dans :

```text
/artifacts
```

Le service `serve` utilise le même volume :

```text
./artifacts:/artifacts
```

La dépendance entre les services est configurée afin que `serve` attende la réussite de `train` :

```yaml
depends_on:
  train:
    condition: service_completed_successfully
```

## Résultats

| Élément | Résultat |
|---|---|
| Service `train` | Code 0 |
| Service `serve` | Fonctionnel |
| Volume `/artifacts` | Fonctionnel |
| Modèle `model.joblib` | Environ 0.5 MB |
| `/health` | 200 OK |

Le pipeline est donc reproductible à partir d'un dossier `artifacts` vide.

Pour lancer le projet :

```powershell
docker compose up -d
```

Pour vérifier les conteneurs :

```powershell
docker compose ps -a
```

Pour arrêter les services :

```powershell
docker compose down
```

---

# 12. Vérification de la reproductibilité

Afin de tester le scénario où aucun modèle préexistant n'est disponible :

```powershell
docker compose down
```

Puis supprimer les artefacts :

```powershell
Remove-Item artifacts\* -ErrorAction SilentlyContinue
```

Relancer :

```powershell
docker compose up -d
```

Vérifier :

```powershell
docker compose ps -a
```

Puis :

```powershell
Invoke-RestMethod http://localhost:8000/ready
```

Le service `train` doit terminer correctement avant que `serve` utilise le modèle généré.

---

# 13. Image Docker publiée

L'image optimisée a été publiée sur GitHub Container Registry :

```text
ghcr.io/ssabiridrissi-rgb/adult-serve:opt6
```

Elle peut être récupérée avec :

```powershell
docker pull ghcr.io/ssabiridrissi-rgb/adult-serve:opt6
```

Cette image correspond à la version optimisée utilisée pour le service d'inférence.

---

# 14. Analyse de sécurité avec Trivy

L'analyse de sécurité a été réalisée sur :

```text
adult-serve:opt6
```

avec l'option :

```text
--ignore-unfixed
```

Commande utilisée :

```powershell
trivy image --ignore-unfixed adult-serve:opt6
```

## Résultats

| Niveau | Nombre de vulnérabilités |
|---|---:|
| LOW | 8 |
| MEDIUM | 36 |
| HIGH | 12 |
| CRITICAL | 0 |
| **Total** | **56** |

La répartition obtenue est :

- **40 vulnérabilités** au niveau Debian ;
- **16 vulnérabilités** au niveau des packages Python.

Aucune vulnérabilité **CRITICAL** n'a été détectée avec l'option `--ignore-unfixed`.

---

# 15. Analyse des couches avec Dive

L'image analysée est :

```text
adult-serve:opt6
```

Commande :

```powershell
dive adult-serve:opt6
```

Résultats :

| Élément | Valeur |
|---|---:|
| Efficiency Score | 98 % |
| Taille affichée | 517 MB |
| Espace potentiellement gaspillé | 11 MB |
| Plus grosse couche | 386 MB |

La couche la plus importante est :

```text
COPY /install /usr/local
```

avec environ **386 MB**.

Elle contient principalement les dépendances Python nécessaires à l'application.

---

# 16. Benchmark

Le projet contient également :

```text
benchmark.sh
```

Ce script permet de réaliser les mesures nécessaires à la comparaison des différentes versions de l'image.

Les mesures réalisées ont notamment permis de comparer :

- le cold build ;
- le rebuild ;
- la taille des images ;
- l'impact des différentes optimisations.

Les résultats sont présentés dans la section **Optimisation progressive de l'image Docker**.

---

# 17. Bilan des optimisations

Les différentes optimisations ont permis de réduire fortement la taille de l'image et surtout le temps nécessaire lors des reconstructions après modification du code.

La progression de la taille est :

```text
625.00 MB
      ↓
261.26 MB
      ↓
169.84 MB
      ↓
165.76 MB
      ↓
166.17 MB
```

Le temps de rebuild évolue notamment de :

```text
63.73 s
```

pour la version naïve à :

```text
2.29 s
```

avec le multi-stage build.

L'optimisation la plus importante pour le cache est la séparation entre les dépendances et le code source.

L'utilisation d'une image `slim` apporte quant à elle le gain principal en taille.

---

# 18. Commandes utiles

### Voir les conteneurs

```powershell
docker compose ps -a
```

### Voir les logs

```powershell
docker compose logs
```

### Logs du service d'entraînement

```powershell
docker compose logs train
```

### Logs du service d'inférence

```powershell
docker compose logs serve
```

### Reconstruire les images

```powershell
docker compose build
```

### Rebuild et lancement

```powershell
docker compose up -d --build
```

### Arrêter les services

```powershell
docker compose down
```

### Lister les images

```powershell
docker images
```

---

# 19. GitHub

Dépôt du projet :

```text
https://github.com/ssabiridrissi-rgb/tp2-docker-cloud-computing
```

Image Docker publiée :

```text
ghcr.io/ssabiridrissi-rgb/adult-serve:opt6
```

---

