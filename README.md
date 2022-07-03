Hola que tal amigos, les dejo los comandos que se para el video de GitOps con Gitlab y ArgoCD

## Instalar minikube

[Install Minikube - Kubernetes (k8s-docs.netlify.app)](https://k8s-docs.netlify.app/en/docs/tasks/tools/install-minikube/)

## Instalar ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Gitlab

Primero necesitamos 2 repositorios, uno para la aplicación y otro para la configuración

Repositorio de aplicacion sera una aplicacion de ejemplo de express

el archivo gitlab ci que es donde se configura el pipeline tendrá la siguiente estructura

```yaml
stages:          # List of stages for jobs, and their order of execution
  - build
  - test
  - trigger

docker-build:
  # Use the official docker image.
  image: docker:latest
  stage: build
  services:
    - docker:dind
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE/application:$CI_COMMIT_SHORT_SHA
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD"
  script:
    - docker build --pull -t "$IMAGE_TAG" .
    - docker push "$IMAGE_TAG"

unit-test-job:   # This job runs in the test stage.
  stage: test    # It only starts when the job in the build stage completes successfully.
  script:
    - echo "Running unit tests... This will take about 60 seconds."
    - sleep 60
    - echo "Code coverage is 90%"

    

trigger-job:
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE/application:$CI_COMMIT_SHORT_SHA
  stage: trigger
  trigger:
    project: project
    branch: main
```

y el repo de configuracion sera un repo que contendrá los manifiestos de kubernetes que nos permitirán desplegar nuestra aplicación

tendrá un archivo .gitlab-ci para poder hacer la conexión entre el repositorio de aplicacion y el de configuración el cual tendrá la siguiente estructura

```yaml
image: node:latest
stages:
  - deploy

deploy:
  stage: deploy
  only:
    - pipelines
  script:
    - echo $IMAGE_TAG
    - 'sed -i "s|image:.*|image: $IMAGE_TAG|g" k8s/deployment.yml'
    - cat k8s/deployment.yml | grep "image:"
  after_script:
    - |
      CHANGES=$(git status --porcelain | wc -l)
      if [ "$CHANGES" -gt "0" ]; then
        git status
        git config --global user.name $PIPELINE_USER
        git config --global user.email $PIPELINE_EMAIL
        git add --all
        git commit -m "Update image tag"
        git push http://${GITLAB_USER_LOGIN}:${PERSONAL_TOKEN}@gitlab.com/${CI_PROJECT_PATH}.git HEAD:main -o ci.skip
      fi
```

los manifiestos de kubernetes seran un deployment y un service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: image
        ports:
        - containerPort: 80
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```
