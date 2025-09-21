# Pipeline-GitLab-CI

---

Voici un README clair et structuré basé sur ton document :

---

# README - Projet E-commerce avec CI/CD GitLab et déploiement via Argo CD

## Introduction

Ce projet consiste en une application e-commerce développée en HTML, CSS et JavaScript. Tous les fichiers sont regroupés dans un dossier `WEB`. Le projet utilise Docker pour la conteneurisation avec un `Dockerfile` et une configuration Nginx (`nginx.conf`). De plus, un pipeline CI/CD est mis en place via GitLab et Argo CD pour le déploiement continu.

---

## Architecture du Projet

```
GITLAB-CI-CD/
├── .gitlab-ci.yml
└── WEB/
    ├── index.html
    ├── Dockerfile
    └── nginx.conf
```

* **Dockerfile** : Conteneurise l’application avec Nginx sur Alpine Linux.
* **nginx.conf** : Configuration du serveur web Nginx pour servir l’application.

---

## Configuration Docker

**Dockerfile**

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**nginx.conf**

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ =404;
    }
}
```

---

## GitLab CI/CD

### Structure du pipeline `.gitlab-ci.yml`

Le pipeline est divisé en plusieurs étapes :

* **build** : Construction de l’image Docker.
* **test** : Lancement d’un conteneur temporaire pour tester le serveur avec un curl HTTP.
* **push** : Push de l’image construite vers le registry GitLab.
* **deploy** *(en cours d’ajout)* : Déploiement de l’application (à intégrer).

### Exemple de configuration `.gitlab-ci.yml`

```yaml
stages:
  - build
  - test
  - push

variables:
  IMAGE_NAME: "registry.gitlab.com/pipeline-devops/app-web"
  IMAGE_TAG: "v1.2"
  CONTAINER_NAME: "app-web"

build-image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG ./WEB
  only:
    - main
  tags:
    - Runner-App-02
  interruptible: true

test-image:
  stage: test
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker run -d -p 8080:80 --name $CONTAINER_NAME $IMAGE_NAME:$IMAGE_TAG
    - sleep 5
    - curl -s -o /dev/null -w "%{http_code}" http://localhost:8080 | grep 200 || echo "Test échoué"
  after_script:
    - docker rm -f $CONTAINER_NAME || true
  only:
    - main
  tags:
    - Runner-App-02

push-image:
  stage: push
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin registry.gitlab.com
  script:
    - docker push $IMAGE_NAME:$IMAGE_TAG
  only:
    - main
  tags:
    - Runner-App-02
```

---

## Runner GitLab

* Installation et enregistrement d’un runner Docker.
* Configuration via le fichier `config.toml`.
* Utilisation des tags `Runner-App-02` pour cibler ce runner dans le pipeline.

---

## Installation et Configuration Argo CD

### Prérequis

* Helm installé sur la machine (via script curl).
* Kubernetes configuré.

### Installation Argo CD via Helm

```bash
wget https://github.com/argoproj/argo-helm/releases/download/argo-cd-8.5.3/argo-cd-8.5.3.tgz
tar -zxvf argo-cd-8.5.3.tgz
helm install argocd ./argo-cd -n argocd --create-namespace
```

### Accès à l’interface Argo CD

* Mise en place du port forwarding :

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

* Récupération du mot de passe admin :

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

---

## Repository GitLab pour les Manifests Kubernetes

* Création d’un dépôt dédié aux manifests.
* Gestion des clés SSH pour sécuriser l’accès et les opérations Git.
* Ajout de la clé publique dans GitLab.
* Configuration de la variable GitLab CI/CD `GITLAB_SSH_PRIVE`.
* Configuration de l’agent SSH sur la VM pour utiliser la clé privée.

---

## Configuration Argo CD avec GitLab

* Ajout du repository GitLab dans Argo CD via l’interface.
* Création et gestion d’applications Argo CD pour déployer automatiquement les manifests du projet.

---

## Conclusion

Ce projet permet de mettre en place une chaîne complète DevOps, depuis la création d’une application front-end simple, sa conteneurisation, la mise en place d’un pipeline CI/CD dans GitLab, jusqu’au déploiement automatisé avec Argo CD sur Kubernetes.

---

Si tu veux, je peux aussi te faire une version plus courte, ou plus orientée usage. Tu veux ?
