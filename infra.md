
# OS
| 항목              | OS                  | 설명                                                                                               |
| --------------- | ------------------- | ------------------------------------------------------------------------------------------------ |
| host machine os | esxi 6.7 hipervisor | - 온프레미스 머신에  대한 lv1 hypervisor <br>- vm의 생성 및 하드웨어 리소스 관리<br>- vm 과 외부 네트워크 인터페이스간 네트워크 관리 <br> |
| vm              | Ubuntu 22.04.3 LTS  | - 실제 k8s 머신을 실행시키는 vm                                                                            |

# VM 하드웨어 스펙


![vms.png](images%2Fvms.png)

| VM       | role          | hardware specs                                   | 내부망 주소         | 특이사항                                                                               |
| -------- | ------------- | ------------------------------------------------ | -------------- | ---------------------------------------------------------------------------------- |
| control1 | control-plain | - CPU : 4vCPU<br>- RAM  : 4GB<br>- SSD : 400GB   | 192.168.35.94  | - NoSchedule 설정                                                                    |
| data1    | data node     | - CPU : 8vCPU<br>- RAM  : 8GB<br>- SSD : 800GB   | 192.168.35.24  | - dedicated mysql statefulset and pv                                               |
| data2    | data node     | - CPU : 10vCPU<br>- RAM  : 16GB<br>- SSD : 800GB | 192.168.35.63  | - dedicated ES statefulset and pv<br><br>- add more ram to run vector search of ES |
| data3    | data node     | - CPU : 8vCPU<br>- RAM  : 8GB<br>- SSD : 800GB   | 192.168.35.239 | - dedicated grafana and pv<br><br>                                                 |



# Network info

![network.png](images%2Fnetwork.png)


## 전반적인 network 흐름

dns proxy 로는 cloudfront 를 쓰고 있고, 홈 네트워크로 인입된 http(s) 요청들은 dns name 에 의해  k8s nginx-controller에 의해서 분기처리 됩니다.

### 주소

주소 suffix : dev-wiki.dev

### cloud front로 관리되는 호스트 서비스
cloud front로 외부로 노출한 서비스의 구성은 아래와 같습니다


| 이름             | 환경    | 주소                                                          | 설명                                                                                 |
| -------------- | ----- | ----------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| 서비스 도메인        | prod  |                                                             | 서비스 중인 js 페이지의 도메인입니다.                                                             |
|                | dev   |                                                             | 서비스 중인 js 페이지의 도메인입니다.                                                             |
| web api 문서     | prod  | https://webapi.dev-wiki.dev/swagger-ui/index.html           | 리소스의 생성, 변경 등의 로직을 담당하는 api 서버 문서입니다.<br>- swagger 로 작성됨                           |
|                | dev   | https://developwebapi.dev-wiki.dev/swagger-ui/index.html    | 위와 동일                                                                              |
| search api 문서  | prod  | https://searchapi.dev-wiki.dev/swagger-ui/index.html        | 리소스에 대한 검색 api 입니다. 검색 이외에도 유사 문서 추천, 인기 문서 리스트 등의 api 를 포함합니다.<br>- swagger 로 작성됨 |
|                | dev   | https://developsearchapi.dev-wiki.dev/swagger-ui/index.html | 위와 동일                                                                              |
| grafana        | 구분 없음 | https://grafana.dev-wiki.dev/login                          | - 로그검색<br>- 클러스터 모니터링 <br><br>의 역할을 합니다. guest 용 계정은 각자 담당자분께 메일로 보내드렸습니다.<br>     |
| parsing lab    | 구분 없음 | https://parsingbot.dev-wiki.dev/lab                         | 번역 컨텐츠 자동 생성용 웹페이지 입니다.<br>자세한 사용법은 페이지의 TODO를 확인해주세요 .                            |
| argocd         | 구분 없음 | https://argocd.dev-wiki.dev/applications                    | - pod deploy 관리                                                                    |
| Elasticsesarch | 구분 없음 | - 주소 비공개 -                                                  | - 엘라스틱 서치                                                                          |
| kibana         | 구분 없음 | https://kibana.dev-wiki.dev/applications                    | - ES 모니터링,                                                                         |


# 기타 주의 사항

## Cloudfront dns proxy 가 https 설정하는 것을 막아야 하는 경우

