> 이 문서는 react 템플릿 프로젝트의 CI/CD에 대한 설명을 포함하고 있습니다.

## 목차
- [목차](#목차)
- [## gitlab-ci.yml](#-gitlab-ciyml)
- [CI/CD Variables 설정](#cicd-variables-설정)
- [Helm Chart Setting](#helm-chart-setting)
- [- `values.yaml`파일 수정](#--valuesyaml파일-수정)
- [- name: gitlab-registry](#--name-gitlab-registry)

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
    - export DEPLOYS=$(helm ls | grep react-app | wc -l)
    # namespace가 없을 경우(react-app)
    # helm chart로 배포
    - if [ ${DEPLOYS}  -eq 0 ]; then helm install "$CI_PROJECT_NAME" . ; else helm upgrade "$CI_PROJECT_NAME" . ; fi
    # namespace가 있을 경우
    #- if [ ${DEPLOYS}  -eq 0 ]; then helm install "$CI_PROJECT_NAME" . --namespace=${STAGING_NAMESPACE}; else helm upgrade "$CI_PROJECT_NAME" . --namespace=${STAGING_NAMESPACE}; fi
```

## CI/CD Variables 설정
- 배포대상 K8s의 config 파일을 복사
```bash
cat ~/.kube/config | base64 | pbcopy`
```
- `Settings > CI/CD` 진입 하여 `Variables`를 expand
- 복사한 config 값을 `KUBE_CONFIG`로 add

## Helm Chart Setting
- 프로젝트 root 디렉토리에서 helm chart 생성(템플릿 프로젝트에서는 helm-chart로 생성함)
`helm create <helm chart>`
- `values.yaml`파일 수정
```yaml
image:
  repository: ${CI_REGISTRY_IMAGE}
  pullPolicy: IfNotPresent
  tag: latest

imagePullSecrets:
  - name: gitlab-registry
```

