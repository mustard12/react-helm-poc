> 이 문서는 react 템플릿 프로젝트를 Helm Chart를 이용한 배포 CI/CD에 대한 설명을 포함하고 있습니다.

## 목차
- [목차](#목차)
- [버젼 정보](#버젼-정보)
- [Build](#build)
- [Test](#test)
- [Docker Build](#docker-build)
- [Deploy](#deploy)
  - [CI/CD Variables 설정](#cicd-variables-설정)
  - [Helm Chart Setting](#helm-chart-setting)

## 버젼 정보
- helm v1.20
- kubectl v3.5.3

## gitlab-ci.yml 구성
- stage를 명시화
- CI/CD 파이프라인을 수행하기 위한 variables를 입력
- build, test, docker-build, deploy job으로 구성

## Build
- npm을 사용할 수 있는 node image를 사용하여 build
```yaml
build:
  stage: build
  image: node
  script: 
    - echo "Start building App"
    - npm install
    - npm build
    - echo "Build successfully!"
  artifacts:
    expire_in: 1 hour
    paths:
      - build
      - node_modules/
```

## Test
- Build job과 마친가지로 npm으로 사전에 정의된 test 수행
```yaml
test:
  stage: test
  image: node
  script:
    - echo "Testing App"
    - CI=true npm test
    - echo "Test successfully!"
```

## Docker Build
- 프로젝트 repository root에 있는 dockerfile로 docker image build 수행
- 파이프라인을 작동시킨 user의 정보로 docker image를 gitlab에 해당 project container registry로 push
  - `Packages & Registries > Container Registry` 메뉴에서 image를 확인 가능

```yaml
docker-build:
  stage: docker-build
  image: docker:latest
  services: 
    - name: docker:19.03.8-dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - docker build --pull -t "$CI_REGISTRY_IMAGE" .
    - docker push "$CI_REGISTRY_IMAGE"
```

## Deploy
- docker-build job에서 생성된 image를 helm chart를 이용하여 k8s로 배포
```yaml
deploy:
  stage: deploy
  image: lwolf/helm-kubectl-docker:latest
  before_script:
    - mkdir -p /etc/deploy
    # kubernetes config 설정
    - echo ${KUBE_CONFIG} | base64 -d > ${KUBECONFIG}
    # gitlab-registry 이름으로 imagePullSecretes 생성(values.yaml 참고)
    - kubectl create secret docker-registry gitlab-registry --docker-server="$CI_REGISTRY" --docker-username="$CI_REGISTRY_USER" --docker-password="$CI_REGISTRY_PASSWORD" --docker-email="$GITLAB_USER_EMAIL" -o yaml --dry-run=client | kubectl apply -f -
  script:
    - cd helm-chart
    - export API_VERSION="$(grep "appVersion" Chart.yaml | cut -d" " -f2)"
    - export DEPLOYS=$(helm ls | grep "$CI_PROJECT_NAME" | wc -l)
    # namespace가 없을 경우(react-app)
    # helm chart로 배포
    - if [ ${DEPLOYS}  -eq 0 ]; then helm install "$CI_PROJECT_NAME" . ; else helm upgrade "$CI_PROJECT_NAME" . ; fi
    # namespace가 있을 경우
    #- if [ ${DEPLOYS}  -eq 0 ]; then helm install "$CI_PROJECT_NAME" . --namespace=${STAGING_NAMESPACE}; else helm upgrade "$CI_PROJECT_NAME" . --namespace=${STAGING_NAMESPACE}; fi
```


### CI/CD Variables 설정
- 배포대상 K8s의 config 파일을 복사
```bash
cat ~/.kube/config | base64 | pbcopy`
```
- `Settings > CI/CD` 진입 하여 `Variables`를 expand
- 복사한 config 값을 `KUBE_CONFIG`로 add

### Helm Chart Setting
- 프로젝트 root 디렉토리에서 helm chart 생성(템플릿 프로젝트에서는 helm-chart로 생성함)
```bash
helm create $helm_chart_name
```
- `values.yaml`파일 수정
```yaml
image:
  repository: <CI_REGISTRY_IMAGE>
  pullPolicy: IfNotPresent
  tag: latest

imagePullSecrets:
  - name: gitlab-registry
````

## gitlab-ci.yml
```yaml
stages:
  - build
  - test
  - docker-build
  - deploy

variables:
  CONTAINER_IMAGE: ${CI_REGISTRY}/${CI_PROJECT_PATH}:${CI_BUILD_REF_NAME}_${CI_BUILD_REF}
  CONTAINER_IMAGE_LATEST: ${CI_REGISTRY}/${CI_PROJECT_PATH}:latest
  DOCKER_DRIVER: overlay2

  KUBECONFIG: /etc/deploy/config
  STAGING_NAMESPACE: app-stage
  PRODUCTION_NAMESPACE: app-prod

build:
  stage: build
  image: node
  script: 
    - echo "Start building App"
    - npm install
    - npm build
    - echo "Build successfully!"
  artifacts:
    expire_in: 1 hour
    paths:
      - build
      - node_modules/

test:
  stage: test
  image: node
  script:
    - echo "Testing App"
    - CI=true npm test
    - echo "Test successfully!"

docker-build:
  stage: docker-build
  image: docker:latest
  services: 
    - name: docker:19.03.8-dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - docker build --pull -t "$CI_REGISTRY_IMAGE" .
    - docker push "$CI_REGISTRY_IMAGE"
  
deploy:
  stage: deploy
  image: lwolf/helm-kubectl-docker:latest
  before_script:
    - mkdir -p /etc/deploy
    # kubernetes config 설정
    - echo ${KUBE_CONFIG} | base64 -d > ${KUBECONFIG}
    # gitlab-registry 이름으로 imagePullSecretes 생성(values.yaml 참고)
    - kubectl create secret docker-registry gitlab-registry --docker-server="$CI_REGISTRY" --docker-username="$CI_REGISTRY_USER" --docker-password="$CI_REGISTRY_PASSWORD" --docker-email="$GITLAB_USER_EMAIL" -o yaml --dry-run=client | kubectl apply -f -
  script:
    - cd helm-chart
    - export API_VERSION="$(grep "appVersion" Chart.yaml | cut -d" " -f2)"
    - export DEPLOYS=$(helm ls | grep "$CI_PROJECT_NAME" | wc -l)
    # namespace가 없을 경우(react-app)
    # helm chart로 배포
    - if [ ${DEPLOYS}  -eq 0 ]; then helm install "$CI_PROJECT_NAME" . ; else helm upgrade "$CI_PROJECT_NAME" . ; fi
    # namespace가 있을 경우
    #- if [ ${DEPLOYS}  -eq 0 ]; then helm install "$CI_PROJECT_NAME" . --namespace=${STAGING_NAMESPACE}; else helm upgrade "$CI_PROJECT_NAME" . --namespace=${STAGING_NAMESPACE}; fi
```

## 참고
- [Multiple-stage Kubernetes deployments with GitLab and Kustomize](https://blog.codecentric.de/en/2019/11/multple-stage-kubernetes-deployments-with-gitlab-and-kustomize/)
- [Deploy node.js app to kubernetes using helm](https://medium.com/@cloudegl/run-node-js-app-using-kubernetes-helm-bb87747785a)
- [How to create a CI/CD pipeline with Auto Deploy to Kubernetes using GitLab and Helm](https://about.gitlab.com/blog/2017/09/21/how-to-create-ci-cd-pipeline-with-autodeploy-to-kubernetes-using-gitlab-and-helm/)
