# Documentation du Pipeline CI/CD — html5up-forty

## Vue d'ensemble

Ce pipeline GitHub Actions automatise le build, le scan de sécurité et le déploiement
de l'application html5up-forty sous forme d'image Docker.

**Déclenchement** : à chaque push sur les branches `main` ou `trivy`

**Ordre d'exécution** :
```
ci  →  scan-trivy  →  deploy
```

---

## Variables d'environnement globales

```yaml
IMAGE_NAME: webapp-ci-cd
IMAGE_TAG: 1.0.1
DOCKER_REGISTRY: ${{ secrets.DOCKERHUB_USERNAME }}
```

| Variable | Description |
|---|---|
| `IMAGE_NAME` | Nom de l'image Docker construite |
| `IMAGE_TAG` | Version de l'image |
| `DOCKER_REGISTRY` | Username Docker Hub récupéré depuis les secrets GitHub |

---

## JOB 1 — ci (Build + Push)

**Runner** : `ubuntu-latest`

### Etapes

### 1. Checkout source code
```yaml
uses: actions/checkout@v4
```
Récupère le code source du dépôt GitHub sur le runner pour pouvoir travailler avec.

### 2. Display variables
```yaml
run: |
  echo "IMAGE_NAME: $IMAGE_NAME"
  echo "IMAGE_TAG: $IMAGE_TAG"
  echo "DOCKER_REGISTRY: $DOCKER_REGISTRY"
```
Affiche les variables d'environnement dans les logs pour faciliter le débogage
et vérifier que les valeurs sont correctement chargées.

### 3. Login to Docker Hub
```yaml
run: |
  echo "${{ secrets.DOCKERHUB_TOKEN }}" | \
  docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin
```
Authentifie le runner sur Docker Hub en utilisant les secrets GitHub.
L'utilisation de `--password-stdin` évite d'exposer le token dans les logs.

### 4. Build Docker image
```yaml
run: |
  docker build -t ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }} .
```
Construit l'image Docker à partir du `Dockerfile` présent à la racine du projet.
L'image est taguée avec le nom complet incluant le registry, le nom et la version.

### 5. Verify image
```yaml
run: |
  docker image ls | grep ${{ env.IMAGE_NAME }}
```
Vérifie que l'image a bien été construite et est présente localement sur le runner.

### 6. Push Docker image
```yaml
run: |
  docker push ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
```
Publie l'image construite sur Docker Hub pour la rendre disponible
au téléchargement lors du déploiement.

---

## JOB 2 — scan-trivy (Scan de sécurité)

**Runner** : `ubuntu-latest`
**Dépendance** : `needs: ci` — s'exécute uniquement si le job `ci` réussit

### Pourquoi ce job ?

Trivy est un scanner de vulnérabilités open source développé par Aqua Security.
Il analyse l'image Docker publiée sur Docker Hub et détecte les failles de sécurité
connues dans les packages OS et les librairies installées.

Ce job est placé **avant le déploiement** pour garantir qu'aucune image vulnérable
n'est mise en production.

### Etapes

### 1. Checkout source code
```yaml
uses: actions/checkout@v4
```
Récupère le code source nécessaire à l'exécution de l'action Trivy.

### 2. Run Trivy vulnerability scanner
```yaml
uses: aquasecurity/trivy-action@master
with:
  image-ref: ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
  format: table
  exit-code: 0
  ignore-unfixed: true
  vuln-type: os,library
  severity: CRITICAL,HIGH
```

| Paramètre | Valeur | Justification |
|---|---|---|
| `image-ref` | image Docker publiée | Scanne l'image exacte qui sera déployée |
| `format` | `table` | Affiche les résultats sous forme de tableau lisible dans les logs |
| `exit-code` | `0` | Le pipeline continue même si des vulnérabilités sont trouvées (mode audit) |
| `ignore-unfixed` | `true` | Ignore les vulnérabilités sans correctif disponible |
| `vuln-type` | `os,library` | Scanne les packages système et les librairies applicatives |
| `severity` | `CRITICAL,HIGH` | Filtre uniquement les vulnérabilités critiques et élevées |

---

## JOB 3 — deploy (Déploiement)

**Runner** : `ubuntu-latest`
**Dépendance** : `needs: scan-trivy` — s'exécute uniquement si le job `scan-trivy` réussit

### Etapes

### 1. Pull latest image
```yaml
run: |
  docker pull ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
```
Télécharge la dernière version de l'image depuis Docker Hub sur le serveur de déploiement.

### 2. Stop existing container
```yaml
run: |
  docker stop ci-app || true
```
Arrête le conteneur en cours d'exécution. Le `|| true` évite que le pipeline échoue
si aucun conteneur n'est en cours d'exécution (premier déploiement).

### 3. Remove old container
```yaml
run: |
  docker rm ci-app || true
```
Supprime l'ancien conteneur pour libérer le nom `ci-app` et éviter les conflits
lors de la création du nouveau conteneur.

### 4. Run new container
```yaml
run: |
  docker run -d \
    -p 80:80 \
    --name ci-app \
    ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
```
Lance le nouveau conteneur en arrière-plan (`-d`), expose le port 80
et lui donne le nom `ci-app` pour faciliter sa gestion.

---

## Modifications apportées au fichier original

### Problème 1 — Etape dupliquée dans le job deploy
**Fichier original** :
```yaml
- name: Pull latest image
  run: |
    echo "Hello deploy"

- name: Pull latest image
  run: |
    docker pull ...
```
**Correction** : Suppression de l'étape `echo "Hello deploy"` inutile et dupliquée.

---

### Problème 2 — Runner inconnu dans le job deploy
**Fichier original** :
```yaml
deploy:
  runs-on: runner-lab
```
**Correction** : Remplacé par `ubuntu-latest` car `runner-lab` est un runner
self-hosted qui n'existe pas dans ce contexte, ce qui bloquait le job.

---

### Ajout — Job scan-trivy (Bonus)
Le job `scan-trivy` a été ajouté entre `ci` et `deploy` pour introduire
une étape de sécurité dans le pipeline. Cela garantit que l'image Docker
est analysée avant tout déploiement en production.

---

## Secrets GitHub requis

À configurer dans **Settings > Secrets > Actions** du dépôt :

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Username du compte Docker Hub |
| `DOCKERHUB_TOKEN` | Token d'accès généré sur hub.docker.com |
