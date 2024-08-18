
CI/ CD 개념을 나눠서 설명합니다.
CD pipeline 을 제작을 해놓았으나 운영과정에서 서비스 배포는 다시 수동으로 배포 운영할 예정입니다.


CI는 GithubAction 과 Dockerhub 이용합니다.
자세한 내용은 아래 [CI 링크](#CI) 에서 다룹니다.


CD는 Argocd에서 수행합니다. github에 작성된 k8s  manifest 를 읽어 배포합니다.

---


# 배포 리소스 설명

## 1. Github actions
프로덕트 브랜치에 배포된 gradle 프로젝트를 테스트하고 dockerhub 로 이미지를 배포합니다.

gradle test 에 실패하면 실패 알림 후,  이미지를 배포하지 않습니다.

## 2. Dockerhub
이미지 관리 hub 로 dockerhub 씁니다.

**harbor등 현재 k8s 환경에서 이미지 호스팅  안하는 이유**
운영 머신이 온프레미스라 램 자원 절약하고 싶어서 안씁니다.
이미지를 다운받을 때, harbor 에서 이미지 전체를 램에 가지고 있는 것으로 보임. 램낭비가 너무 심함

## 3. Argocd
어플리케이션 배포 설정 및 파드 관리 툴로 argocd 를 이용합니다.

## 4. k8s manifest
argocd + github action 조합에서는 배포될 파드의 k8s manifest 를 직접 작성하여 커밋합니다.
service.yml, deploy.yml 파일을 작성합니다.

작성 템플릿은 아래 [리소스 템플릿](#리소스 템플릿)을 참고해주세요.

---

# CI

CI 의 흐름에 대한 도식화와 간단한 설명입니다.


주의해야 할 점은 배포 manifest의 문법이나 배포 제약사항에 문제가 있을 경우, CI pipeline 에서는 확인이 불가하기 때문에, 이런 내용은 배포 후 개발환경에서

![ci.png](images%2Fci.png)

1. 레포지토리에 배포 전 service.yml, deploy.yml, github flow script 작성필요 ( [리소스 템플릿](#리소스 템플릿) )

2. 작업 브랜치 배포 후 배포 브랜치로 push, 배포 브랜치는 github flow script 의 branch.on 에 명시 되어 있어야 함.
3. github action 수행, 아래의 작업 실행함
    1. CI 용 vm 및 gradle 환경 초기화
    2. gradle test and build
    3. docker image 생성
    4. dockerhub deploy
4. 위 github action 작업이 완료되면 종료

   
---

## CD

CD pipeilne 의 과정은 아래와 같으나 보통 수동배포로 많이 할 예정입니다.
(프론트랑 합을 맞춰야 하는 경우가 있어서 수동이 오히려 편해요)

CD 환경은 거의 Argocd 가  제공해주는 기능과 옵션으로 돌아갑니다.



![cd.png](images%2Fcd.png)


1. 배포 전 아래의 작업이 선행되어 있어야 합니다.
    1. docker hub 에 배포된 이미지가 배포되어 있어야 합니다.
    2. github repo 에 k8s deploy , service 마니페스트가 작성되어 있어야 합니다.
    3. docker hub, repo 에 대한 접근 권한이 K8s secret 으로 등록이 되어 있어야 합니다.

3. argocd 기본 동작을 따릅니다. 확인 주기는 기본과 다를 수 있습니다.
    1. 3분에 한번 argocd에서 읽어온 repo 형상과 원격 저장소의 repo 형상이 다른지 확인합니다.
    2. 만약 다르다면 pull 해서 코드를 읽어옵니다. 해당 레포엔 배포용 k8s manifest 가 있어야 합니다.
    3. 배포 manifest 를 실행 합니다. 이 배포 manifest 의 내용에 따라  명시된 주소의 dockerhub 에서 docker image 를 가져옵니다.


### 배포 주의 사항

**주의 사항: 배포 버전별 이미지를 별도로 저장하세요**

argocd 가 배포 마니페스트를 수행할 때 `kubectl apply -f ...` 를 그대로 수행합니다.
manifest 명령어를 처리하기 위한 docker pull 시 옵션도 기본 옵션으로 수행하는데

`docker pull {repo}:{tag}`

의 기본 옵션은 같은 `{repo}:{tag}` 가 있으면 다운로드 받지 않고 기존에 가지고 있는 이미지를 사용합니다.
만약 배포한다면 태그를 버전별로 다르게 두어야 합니다.

github action 스크립트 작성 시, 버전 별 태그를 명시해주세요 .

# 리소스 템플릿

배포 과정에 작성해야 하는 리소스의 샘플입니다.


## github flow

{"..."} 안의 내용만 가져다 바꾸시면 됩니다.


```yml
# workflow 명

name: prod environment for {"app name"} |  { "project name"}

run-name: ${{github.actor}} is publish deploy on {"app name"} & prod env
on:
  push:
    branches:
      -  prod/{"app name"}
jobs:
  Explore-Github-Actions:
    runs-on: ubuntu-latest
    steps:
      # print ubuntu & repo info
      - run: echo "initial job echo this sentence ~ "
      - run: echo "event :${{github.event_name}} "
      - run: echo "os = ${{runner.os}} "
      - run: echo "branch = ${{ github.ref }} ."
      - run: echo "repo = ${{ github.repository }}."

      # checkout source s:
      - name: Check out repository code
        uses: actions/checkout@v4
      - run: echo "💡 The ${{ github.repository }} repository has been cloned to the runner."

      # java config -> 21 versin  & temurin
      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          java-version: '21'  # JDK 버전을 21로 설정
          distribution: 'temurin'  # Eclipse Temurin을 사용하는 경우

      # gradle build 하기
      - run: echo "setup gradle"
      - name: Initialize Gradle Project
        run: gradle init --warning-mode all --overwrite

      - name: Build with Gradle
        uses: gradle/gradle-build-action@v2
        with:
          gradle-version: 8.7
          arguments: '{"module location of gradle project"}:bootJar'

      # 패키징 된 jar 파일 app.jar 로 옮기기
      - name: Copy Jar file
        run: mv {"project location of gradle project"}/build/libs/$(ls {"project location of gradle project"}/build/libs) app.jar

      -
        name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: {"docker hub repo"}

      # try login docker hub
      - name: log in to docker hub
        uses: docker/login-action@v3
        with:
          username: ${{secrets.DOCKER_USERNAME}}
          password: ${{secrets.DOCKER_PASSWORD}}
      - run: echo "docker hub logged in ~ "

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile                     #빌드 시 사용할 Dockerfile의 위치 지정
          platforms: linux/amd64                 #이미지 등록 시 Platform 이름 지정
          push: true
          tags: {"docker hub repo"}:{"image tag"}
          labels: ${{ steps.meta.outputs.labels }}
```


## 파드 디플로이 샘플


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  앱 이름 
  namespace: 네임스페이스 이름 
spec:
  replicas: 2
  selector:
    matchLabels:
      app:  앱 이름 - 서비스 연결용 
  template:
    metadata:
      labels:
        app:  앱 이름 - 서비스 연결용 
    spec:
      containers:
        - name:  컨테이너 이름 
          image: 도커 이미지 
          imagePullPolicy: Always
          ports:
            - containerPort: 8000

		  # 필요한 파일 리소스 가져오기  (아래 볼륨 등록을 여기서 pod 의 경로로 지정 )
          volumeMounts:
            - name: cert-volume
              mountPath: /etc/certs

            - name: prod-ca-cer
              mountPath: /etc/certs/ca

            - name: prod-server-cer
              mountPath: /etc/certs/server

		 # 필요한 ENV 등록하기 
          env:
            - name: ES_SEARCH_ACCOUNT
              valueFrom:
                secretKeyRef:
                  name: develop-search-api-env
                  key: ES_SEARCH_ACCOUNT

            - name: ES_SEARCH_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: develop-search-api-env
                  key: ES_SEARCH_PASSWORD

            - name: ES_API_KEY
              valueFrom:
                secretKeyRef:
                  name: develop-search-api-env
                  key: ES_API_KEY


          resources:
            limits:
              cpu: 필요한 만큼
              memory: 필요한 만큼

		 # 스프링부트인 경우 여기에서 profile 설정 
          args: ["--spring.profiles.active=dev"]

	  # 볼륨 등록-> pod 생성시 필요한 파일 업로드 
      volumes:
        - name: cert-volume
          secret:
            secretName: prod-es-cer

        - name: prod-ca-cer
          secret:
            secretName: prod-ca-cer

        - name: prod-server-cer
          secret:
            secretName: prod-server-cer

	  # docker hub login 정보 여기서 명시 
      imagePullSecrets:
        - name: dockerhubsecret
```