# Pipeline-GitLab-CI

---

<img width="2334" height="987" alt="image" src="https://github.com/user-attachments/assets/39fe3174-d2ee-47bc-97c7-52440cedeb11" />


---

# Projet E-commerce avec CI/CD GitLab et déploiement via Argo CD

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
  - deploy

variables:
  IMAGE_NAME: "registry.gitlab.com/pipeline-devops/app-web"
  IMAGE_TAG: "v3.0"
  CONTAINER_NAME: "app-web"

# Étape 1 : Build
build-image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - echo "Construction de l'image Docker..."
    - docker build -t $IMAGE_NAME:$IMAGE_TAG ./WEB
  only:
    - main
  tags:
    - Runner-App-02
  interruptible: true

# Test

test-image:
  stage: test
  image: docker:latest
  services:
    - docker:dind
  script:
    - echo "Lancement du conteneur temporaire..."
    - docker run -d -p 8080:80 --name $CONTAINER_NAME $IMAGE_NAME:$IMAGE_TAG
    - echo "Attente que NGINX démarre..."
    - sleep 5
    - echo "Test HTTP avec curl"
    - curl -s -o /dev/null -w "%{http_code}" http://localhost:8080 | grep 200 || echo "Le test a échoué, mais on continue"
  after_script:
    - docker rm -f $CONTAINER_NAME || true
  only:
    - main
  tags:
    - Runner-App-02

# Étape 3 : Push
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
  # Deploy App
deploy-app:
  stage: deploy
  image: ubuntu:22.04
  before_script:
    - apt-get update -y && apt-get install -y openssh-client git curl
    - curl -L https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -o /usr/bin/yq
    - chmod +x /usr/bin/yq
    - mkdir -p $HOME/.ssh
    - echo "$GITLAB_SSH_PRIVE" > $HOME/.ssh/id_rsa
    - chmod 600 $HOME/.ssh/id_rsa
    - ssh-keyscan -H gitlab.com >> $HOME/.ssh/known_hosts
    - chmod 644 $HOME/.ssh/known_hosts
    - eval $(ssh-agent -s)
    - ssh-add $HOME/.ssh/id_rsa
    - git config --global user.email "gitlab-ci@cloud-devops.com"
    - git config --global user.name "gitlab-ci"
    - git clone git@gitlab.com:pipeline-devops/helm-app.git
    - cd helm-app
  script:
    - yq eval ".image.tag = \"${IMAGE_TAG}\"" -i web-app/values.yaml
    - git add web-app/values.yaml
    - |
      if git diff --cached --quiet; then
        echo "Aucune modification détectée, rien à committer."
      else
        git commit -m "Update image tag to ${IMAGE_TAG}"
        git push
      fi
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
<img width="1722" height="650" alt="image" src="https://github.com/user-attachments/assets/4eb2019b-1e08-4e7e-9aba-44d7c297dd36" />

---


## Configuration Argo CD avec GitLab

* On ajoute le repository GitLab dans Argo CD via l’interface.
* Création et gestion d’applications Argo CD pour déployer automatiquement les manifests du projet.
* Donc lors du changment de ce repository argo va mettre à jour notre application sur notre cluster k8S
---



## Conclusion

Ce projet permet de mettre en place une chaîne complète DevOps, depuis la création d’une application front-end simple, sa conteneurisation, la mise en place d’un pipeline CI/CD dans GitLab, jusqu’au déploiement automatisé avec Argo CD sur Kubernetes.

---
OUSMANE KA

