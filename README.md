# TP2 Docker / Cloud Computing

Projet de conteneurisation d'un pipeline d'entrainement et d'une API FastAPI d'inference.

## Prerequis

- Docker avec Docker Compose

## Construire les images

Depuis la racine du depot :

```powershell
docker build -f Dockerfile.train -t adult-train:naif .
docker build -f Dockerfile.serve.opt -t adult-serve:opt6 .
```

Compose utilise ces images locales. Il ne les construit pas automatiquement.

## Lancer les services

```powershell
docker compose up
```

L'API est accessible sur `http://localhost:8000`. Sa documentation interactive est sur `/docs`, et la sonde de sante sur `/health`.

Arreter les services :

```powershell
docker compose down
```

Le dossier `artifacts/` contient le modele et les metriques. Compose le monte dans les services afin de partager les artefacts.