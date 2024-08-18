
# 소개

엔지니어링 관련 문서의 링크 모음집입니다.

내용별로 별개의 문서로 나누어 설명합니다. 


## 1. 인프라

아래의 링크에서 운영환경에 대해 다룹니다.

- 하드웨어 스펙
- k8s 운영환경 
- 네트워크 설정 

바로가기 : [인프라 보기 링크](infra.md)


## 2. 서비스 아키택쳐(백엔드)

대략적인 서비스 아키택쳐에 대해 다룹니다. 

- 백엔드 아키택쳐
- 역할과 pod 갯수
- ingress 되고 있는 주소와 서비스 리스트

바로가기 : [서비스아키택쳐](service_architecture.md)

## 3. 자바 코드 관리와 코드 템플릿 

gradle multimodule 과 각 모듈/서비스 별 분리/테스트에 대해 다룹니다.

코드를 공개하기는 곤란해서 벡엔드 맴버가 정리한 룰 정리로 대체합니다.

- 멀티모듈 환경 구성 
- interface rule ( 역할 분리 규칙 ) 
- test 규칙과 best practice 

바로가기 : [백엔드 코드 룰](https://github.com/dkGithup2022/spring-boot-multimodule-template-example)


## 3. 검색 서비스 

검색 서비스에 대해 대해 다룹니다. 

- bm25 기반 검색에서 analzyer 와 sorting script 
- vector search 를 위한 파이프라이닝 구조 
- ~~A/B 테스트 구조와 웹로깅 ( TODO )~~

바로가기 : [검색 서비스](search.md)

## 4. 큐 

비동기 요청처리를 위한 큐에 대해 다룹니다. 

as-is, to-be 퍼포먼스 비교와 적용사례, api에 대해 다룹니다.

바로가기 :[병수씨 작성]()

## 기타 기능들 

카테고리화하기 어려운 다른 기술적 내용들 입니다.

#### 1. llm 문서 자동 번역 

서비스 초기 리소스 자동 생성 용으로 만든 내부서비스 입니다.

링크 : https://parsingbot.dev-wiki.dev/lab

#### 2. 계층형 댓글 정리 

계층형 댓글, be 에서 커서기반 페이지 api 를 제공하기 위한 구조에 대한 정리입니다.

링크 : [바로가기]()


#### 3. distributed lock 정리 

redis client module 의 내부 기능으로서 Spel 에 대응하는 작업에 대한 분산락을 제공합니다.

링크 : [바로가기]()

#### 4. 
