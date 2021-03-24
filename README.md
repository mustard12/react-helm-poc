> 이 문서는 react 템플릿 프로젝트의 CI/CD에 대한 설명을 포함하고 있습니다.

## 목차
- [버전 정보](# 버전 정보)

## 버전 정보
- npm : 7.5.3

## gitlab-ci.yml
```
stages:
  - build
  - test
  - docker-build

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

``` 
