# Week4 — Docker / CI-CD / Kubernetes / FastAPI / Kafka / SQS / Redis / 아키텍처 정리

## 4주차

아래를 중심으로 정리했다.

- Docker / Image / Container / Volume / Daemon
- Jenkins / Argo CD / GitOps / values.yaml / 이미지 태그
- application-cloud.yml 해석
- MySQL / RDS 차이
- FastAPI / REST API 차이
- Redis 개념
- Kafka / SQS / Pub-Sub
- Topic / Partition / Consumer / Consumer Group / Offset
- Kubernetes 핵심 용어
- Cluster / Namespace / Pod / Deployment / Service
- Helm chart / values.yaml / 템플릿 구조
- 도시락 비유로 전체 흐름 이해
- 위 개념들이 소프트웨어 아키텍처와 어떻게 연결되는지

---

## 1. 이번 주 전체 큰 그림

이번 주는 단순히 코드 한 줄이 아니라  
**백엔드 코드가 어떻게 포장되고 배포되고 운영되는지**를 처음부터 끝까지 이해하는 주차였다.

전체 흐름은 아래처럼 볼 수 있다.

```text
코드 작성
→ Dockerfile로 이미지 생성
→ Jenkins가 테스트 / 빌드 / 이미지 푸시
→ GitOps 방식으로 values.yaml 수정
→ Argo CD가 Git 변경 감지
→ Kubernetes에 실제 배포
→ Service를 통해 통신
````

핵심:

* 코드만 있다고 서비스가 끝나는 것이 아님
* 실행 환경 / 배포 / 설정 / 운영 구조까지 이어서 봐야 전체가 이해됨

---

## 2. 도시락 비유로 전체 구조 이해하기

이번 주 개념은 도시락 비유로 보면 가장 이해하기 쉬웠다.

구성:

* 소스코드 = 요리 레시피
* Dockerfile = 도시락 포장 조리법
* Docker image = 포장 완료된 밀키트 / 도시락 원본
* Container = 실제로 전자레인지 돌려 먹는 도시락 1개
* Jenkins = 주방 매니저
* ECR = 포장된 도시락 창고
* values.yaml = 이번에 매장에 올릴 도시락 버전표
* GitOps = “무슨 도시락을 매장에 둘지”를 Git 문서로 관리하는 방식
* Argo CD = 매장 진열 담당
* Kubernetes = 도시락 매장 전체 운영 시스템
* Pod = 도시락 1세트 상자
* Deployment = 도시락 몇 개 둘지 관리하는 점장
* Service = 손님이 보는 대표 주문 창구

핵심:

* Docker는 포장
* Jenkins는 자동화
* GitOps는 배포 버전 기록
* Argo CD는 반영
* Kubernetes는 운영

---

## 3. Docker 핵심 개념

### 3-1. Dockerfile / Image / Container 관계

관계는 아래처럼 이해하면 된다.

```text
Dockerfile → Image → Container
```

의미:

* Dockerfile = 이미지를 만드는 설명서
* Image = 실행 가능한 포장본
* Container = 그 포장본을 실제로 실행한 상태

비유:

* Dockerfile = 도시락 조리법
* Image = 만들어진 냉동 도시락 세트
* Container = 실제로 먹는 도시락

핵심:

* 이미지는 정적 결과물
* 컨테이너는 실행 중인 동적 상태
* 이미지 하나로 컨테이너 여러 개 생성 가능

---

### 3-2. Dockerfile 실제 코드 해석

```dockerfile
FROM gradle:8.13-jdk17 AS builder
WORKDIR /home/gradle/project
COPY --chown=gradle:gradle . .
RUN gradle build --no-daemon -x test

FROM amazoncorretto:17
VOLUME /tmp
COPY --from=builder /home/gradle/project/build/libs/*.jar app.jar
EXPOSE 8080
ENV SPRING_PROFILES_ACTIVE=cloud
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

위 코드는 두 단계로 나눠 이해하면 쉽다.

#### 1단계: 빌드용 방

```dockerfile
FROM gradle:8.13-jdk17 AS builder
WORKDIR /home/gradle/project
COPY --chown=gradle:gradle . .
RUN gradle build --no-daemon -x test
```

설명:

* Gradle + JDK가 있는 빌드 환경 시작
* 작업 폴더 지정
* 프로젝트 코드를 복사
* 테스트는 생략하고 jar 생성

핵심:

* 여기서는 “실행”보다 “만들기”가 목적
* 즉 jar를 뽑아내는 주방 단계

#### 2단계: 실행용 방

```dockerfile
FROM amazoncorretto:17
VOLUME /tmp
COPY --from=builder /home/gradle/project/build/libs/*.jar app.jar
EXPOSE 8080
ENV SPRING_PROFILES_ACTIVE=cloud
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

설명:

* 실행용 Java 17 환경 시작
* `/tmp` 를 볼륨처럼 사용할 수 있는 지점으로 선언
* 앞 단계에서 만든 jar만 가져옴
* 8080 포트를 사용한다고 표시
* Spring을 cloud 프로필로 실행
* 컨테이너 시작 시 `java -jar /app.jar` 실행

핵심:

* 최종 이미지는 실행에 필요한 것만 담음
* 빌드 도구를 최종 이미지에 넣지 않아 가볍다
* 이 구조를 멀티스테이지 빌드라고 한다

---

### 3-3. 멀티스테이지 빌드 의미

멀티스테이지 빌드는 **빌드 환경과 실행 환경을 분리하는 방식**이다.

이유:

* 빌드 도구까지 최종 이미지에 담지 않아도 됨
* 이미지가 더 가벼워짐
* 실행 환경이 더 깔끔해짐

비유:

* 첫 번째 방 = 주방
* 두 번째 방 = 포장실

즉 주방 기계까지 도시락 상자에 넣는 게 아니라
완성품만 포장하는 구조다.

---

## 4. Daemon / Volume / /tmp

### 4-1. daemon 이란 무엇인가

daemon은 **뒤에서 계속 상주하며 동작하는 프로그램**이다.

비유:

* 건물 관리실 직원
* 계속 켜져 있으면서 필요한 일을 처리하는 사람

Docker에서 대표 daemon:

* `dockerd`

핵심:

* 사용자가 명령어를 입력할 때마다 새로 생기는 것이 아님
* Docker 관련 작업을 뒤에서 계속 관리하는 상주 프로그램

---

### 4-2. volume 이란 무엇인가

volume은 **컨테이너 바깥에 두는 저장공간**이다.

비유:

* 컨테이너 = 호텔방
* 볼륨 = 호텔 창고

핵심:

* 컨테이너는 지워질 수 있어도
* 볼륨에 둔 데이터는 남길 수 있음
* DB 데이터나 오래 유지해야 할 파일 저장에 중요

---

### 4-3. `/tmp` 와 `VOLUME /tmp` 의미

`/tmp` 는 리눅스의 임시 파일 폴더다.

보통 저장되는 것:

* 캐시 파일
* 잠깐 쓰는 임시 데이터
* 중간 처리 파일

Dockerfile의

```dockerfile
VOLUME /tmp
```

의미:

* `/tmp` 를 볼륨 마운트 지점으로 사용할 수 있게 선언
* 임시 저장 경로를 분리해서 관리할 수 있게 함

핵심:

* “무조건 영구 저장”이라는 뜻은 아님
* 해당 경로를 외부 저장소처럼 다룰 수 있게 하는 선언

---

## 5. Jenkins / Argo CD / GitOps

### 5-1. Jenkins란 무엇인가

Jenkins는 **반복 작업을 자동화하는 서버**다.

주요 역할:

* 코드 가져오기
* 테스트 수행
* 빌드 수행
* Docker 이미지 생성
* ECR 푸시
* 배포용 설정 파일 수정
* Argo CD 실행

흐름:

```text
Checkout
→ Test
→ Build
→ Docker Build
→ Push
→ values.yaml 수정
→ Argo CD sync
```

핵심:

* 사람이 하던 배포 전 작업을 대신 해주는 자동화 도구

---

### 5-2. GitOps란 무엇인가

GitOps는 문서가 아니라 **운영 방식**이다.

정의:

* 배포할 상태를 Git에 기록하고
* 실제 서버 상태를 Git에 맞추는 방식

즉:

* values.yaml = Git에 있는 배포 상태 기록
* GitOps = 그 기록을 기준으로 운영하는 방식

핵심:

* 서버에 직접 접속해서 수동 배포하지 않음
* Git이 배포 기준점이 됨
* 이력 추적과 롤백이 쉬워짐

---

### 5-3. Argo CD란 무엇인가

Argo CD는 **Git 상태를 실제 Kubernetes 상태로 맞춰주는 도구**다.

명령 예시:

```bash
argocd app sync
argocd app wait --health --timeout 300
```

의미:

* `sync` = Git에 적힌 상태대로 실제 환경 맞춤
* `wait --health` = 정상 상태가 될 때까지 대기

핵심:

* Jenkins가 준비한 배포 상태를
* Argo CD가 실제 Kubernetes에 반영함

---

## 6. values.yaml 과 이미지 태그

### 6-1. values.yaml 이란 무엇인가

`values.yaml` 은 **이미지를 만드는 파일이 아니라 배포에 사용할 값을 적어놓는 설정 파일**이다.

예시:

```yaml
image:
  repository: springboot
  tag: a1b2c3d4
```

의미:

* 어떤 이미지인지
* 그 이미지의 어떤 버전을 배포할지

핵심:

* Dockerfile은 이미지 생성용
* values.yaml은 배포값 지정용

---

### 6-2. 이미지 태그란 무엇인가

이미지 태그는 **Docker 이미지에 붙는 버전 이름표**다.

예시:

```text
springboot:backend-latest
springboot:a1b2c3d4
```

구성:

* `springboot` = 이미지 이름
* `backend-latest`, `a1b2c3d4` = 태그

핵심:

* 태그는 이미지 버전표
* `latest` 는 최근 버전
* 커밋 해시는 특정 시점 버전을 정확히 가리킴

---

### 6-3. Jenkins가 태그를 두 개 쓰는 이유

예:

* `backend-latest`
* `${GIT_COMMIT}`

이유:

* `latest` → 최근 버전 확인용
* 커밋 해시 → 정확한 배포 버전 추적용

핵심:

* latest만 쓰면 추적이 어려움
* 커밋 해시를 같이 쓰면 특정 배포 버전 확인과 롤백이 쉬움

---

## 7. application-cloud.yml 해석

```yaml
spring:
  config:
    import: optional:file:.env[.properties]
    activate:
      on-profile: cloud

  datasource:
    url: jdbc:mysql://${RDS_HOST}:3306/${DB_NAME}
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: ${RDS_USERNAME}
    password: ${RDS_PASSWORD}

  kafka:
    bootstrap-servers: ${KAFKA_HOST}:9092
    consumer:
      group-id: ${KAFKA_CONSUMER_GROUP_ID}

webhook:
  slack:
    url: ${SLACK_WEBHOOK_URL}

fastapi:
  base-url: ${FASTAPI_URL}

aws:
  sqs:
    queue-url: ${SQS_QUEUE_URL}
```

### 묶어서 이해하기

#### 설정 파일 적용 조건

```yaml
spring:
  config:
    import: optional:file:.env[.properties]
    activate:
      on-profile: cloud
```

의미:

* `.env` 파일이 있으면 읽음
* `cloud` 프로필일 때만 이 설정 적용

핵심:

* 운영 환경 전용 설정 파일

#### DB 설정

```yaml
datasource:
  url: jdbc:mysql://${RDS_HOST}:3306/${DB_NAME}
  driver-class-name: com.mysql.cj.jdbc.Driver
  username: ${RDS_USERNAME}
  password: ${RDS_PASSWORD}
```

의미:

* MySQL DB에 접속
* DB 주소 / 계정 / 비밀번호는 환경변수로 주입

핵심:

* 운영 DB 정보는 코드에 직접 쓰지 않음

#### Kafka 설정

```yaml
kafka:
  bootstrap-servers: ${KAFKA_HOST}:9092
  consumer:
    group-id: ${KAFKA_CONSUMER_GROUP_ID}
```

의미:

* Kafka 서버 주소
* Kafka 소비자 그룹 이름

#### Slack / FastAPI / SQS 설정

```yaml
webhook:
  slack:
    url: ${SLACK_WEBHOOK_URL}

fastapi:
  base-url: ${FASTAPI_URL}

aws:
  sqs:
    queue-url: ${SQS_QUEUE_URL}
```

의미:

* Slack 웹훅 주소
* FastAPI 서버 주소
* SQS 큐 주소

핵심:

* 운영 환경에서 붙는 외부 서비스 주소들을 모아둔 설정

---

## 8. MySQL 과 RDS 차이

### MySQL

* 관계형 데이터베이스 엔진

### RDS

* AWS가 제공하는 관리형 데이터베이스 서비스

즉:

* MySQL = DB 종류
* RDS = 그 DB를 대신 운영해주는 서비스

비유:

* MySQL = 자동차 엔진
* RDS = 렌터카 회사

핵심:

* `jdbc:mysql://...` = MySQL 엔진 사용
* `RDS_HOST` = 그 MySQL 서버를 AWS RDS가 제공

---

## 9. FastAPI / REST API / Redis

### 9-1. FastAPI란 무엇인가

FastAPI는 **Python으로 API 서버를 만드는 프레임워크**다.

역할:

* 예측 서버
* 데이터 처리 서버
* AI 모델 추론 서버

비유:

* 본사(Spring)가 분석실(FastAPI)에 요청 보내는 구조

---

### 9-2. FastAPI 코드 예시

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class PredictRequest(BaseModel):
    equip_id: str
    zone_id: str
    temperature: float
    vibration: float

@app.get("/")
async def root():
    return {"message": "Hello FastAPI"}

@app.post("/api/v1/predict")
async def predict(req: PredictRequest):
    score = req.temperature * 0.6 + req.vibration * 0.4

    if score > 80:
        danger_level = "CRITICAL"
        remain_days = 3
    elif score > 50:
        danger_level = "WARNING"
        remain_days = 10
    else:
        danger_level = "INFO"
        remain_days = 30

    return {
        "equipId": req.equip_id,
        "zoneId": req.zone_id,
        "dangerLevel": danger_level,
        "remainDays": remain_days
    }
```

핵심:

* `FastAPI()` = 서버 시작점
* `@app.get("/")` = GET 요청 처리
* `@app.post(...)` = POST 요청 처리
* `BaseModel` = 입력 형식 검증

---

### 9-3. FastAPI 와 REST API 차이

구분:

* FastAPI = 도구 / 프레임워크
* REST API = API 설계 방식

즉:

* FastAPI는 “무엇으로 만들었는가”
* REST는 “어떻게 설계했는가”

핵심:

* FastAPI로 REST API를 만들 수 있음

---

### 9-4. Redis란 무엇인가

Redis는 **메모리 기반의 매우 빠른 저장소**다.

주 사용처:

* 캐시
* 세션
* 인증 코드
* 임시 토큰

비유:

* Redis = 카운터 위 포스트잇 보드
* MySQL = 회사 공식 장부

핵심:

* Redis는 빠른 조회와 임시 데이터에 강함
* MySQL은 정식 영구 저장에 강함

---

## 10. SQS 와 Kafka

### 10-1. SQS란 무엇인가

SQS는 **작업 요청을 잠깐 넣어두는 AWS 메시지 큐**다.

비유:

* 택배 보관함

흐름:

```text
메시지 넣기
→ 큐에 대기
→ 나중에 소비자가 꺼내 처리
```

핵심:

* 비동기 작업 처리에 적합
* “이 일 좀 처리해줘” 용도에 강함

---

### 10-2. Kafka란 무엇인가

Kafka는 **실시간 이벤트를 빠르게 흘리고 저장하고 여러 소비자가 읽을 수 있는 이벤트 스트리밍 플랫폼**이다.

비유:

* 방송국 + 로그 장부

흐름:

```text
이벤트 발생
→ Topic에 기록
→ 여러 Consumer가 읽음
```

핵심:

* 실시간 이벤트 흐름 처리에 적합
* 저장 / 구독 / 다중 소비에 강함

---

### 10-3. SQS 와 Kafka 차이

#### SQS

* 작업 큐
* 우편함
* “이 작업 해줘”

#### Kafka

* 이벤트 스트림
* 방송 채널
* “이 이벤트가 계속 발생 중이야”

핵심:

* SQS = 작업 요청 전달
* Kafka = 실시간 이벤트 흐름

---

## 11. Pub/Sub 구조

Pub/Sub는 **보내는 쪽과 받는 쪽을 직접 연결하지 않고 중간에 Topic을 두는 구조**다.

구성:

* Producer = 메시지 보내는 쪽
* Topic = 메시지 저장 공간
* Consumer = 메시지 읽는 쪽

비유:

* 유튜버 = Producer
* 채널 = Topic
* 구독자 = Consumer

흐름:

```text
Producer → Topic → Consumer
```

핵심:

* 한 번 발행하면 여러 Consumer가 읽을 수 있음
* 보내는 쪽과 받는 쪽이 느슨하게 연결됨

---

## 12. Kafka 구조 — Topic / Partition / Consumer / Offset

### 12-1. Topic

Topic은 **메시지 게시판**이다.

예:

* `ENVIRONMENT`
* `WEARABLE`
* `ALARM`

핵심:

* 성격이 다른 데이터는 Topic을 나눠서 저장
* Topic 여러 개 사용하는 것이 일반적

---

### 12-2. Partition

Partition은 **Topic 안의 여러 칸**이다.

구조:

```text
Topic
 ├── Partition 0
 ├── Partition 1
 ├── Partition 2
```

의미:

* 병렬 처리 단위
* Topic 하나는 여러 Partition으로 나눌 수 있음

핵심:

* Topic = 큰 통
* Partition = 그 안의 여러 칸

---

### 12-3. Consumer 와 Partition 관계

#### 컨슈머 1개일 때

```text
Consumer A → P0, P1, P2 전부 읽음
```

#### 컨슈머 3개일 때

```text
Consumer A → P0
Consumer B → P1
Consumer C → P2
```

핵심:

* 컨슈머 수에 따라 여러 파티션을 하나가 읽을 수도 있음
* 같은 Consumer Group 안에서는 한 파티션을 동시에 둘이 읽지 않음

---

### 12-4. Consumer Group

Consumer Group은 **같은 목적의 Consumer 묶음**이다.

예:

* 저장용 그룹
* 알림용 그룹

핵심:

* 같은 그룹 안에서는 파티션을 나눠 읽음
* 다른 그룹은 같은 Topic을 다시 읽을 수 있음

---

### 12-5. Offset

Offset은 **파티션 안 메시지의 줄번호**다.

예:

```text
offset 0 : 메시지 A
offset 1 : 메시지 B
offset 2 : 메시지 C
```

비유:

* 책갈피 번호

핵심:

* 어디까지 읽었는지 기록하는 기준
* 재시작 후 이어서 읽는 데 중요

---

## 13. Kubernetes 핵심 구조

### 13-1. Cluster

Cluster는 **Kubernetes 전체 운영 공간**이다.

비유:

* 학교 전체
* 아파트 단지 전체

구성:

* control plane = 관리실
* worker node = 실제 앱이 도는 서버들

---

### 13-2. Namespace

Namespace는 **Cluster 안의 구역 나누기**다.

예:

* dev
* staging
* prod

비유:

* Cluster = 학교 전체
* Namespace = 반

핵심:

* 같은 클러스터 안에서도 리소스를 구분 가능

---

### 13-3. Pod

Pod는 **Kubernetes가 관리하는 가장 작은 배포 단위**다.

쉽게 말하면:

* 컨테이너를 담아 실행하는 상자

보통:

* 컨테이너 1개 = Pod 1개

핵심:

* Docker 핵심은 컨테이너
* Kubernetes 핵심은 Pod

---

### 13-4. Deployment

Deployment는 **Pod 개수와 버전을 관리하는 관리자**다.

역할:

* Pod 몇 개 유지할지 결정
* 죽으면 다시 생성
* 새 버전으로 교체

문서 표현을 쉽게 바꾸면:

* “원하는 상태를 적어두면 실제 상태를 거기에 맞추는 감독관”

---

### 13-5. Service

Service는 **Pod들 앞에 붙는 고정 주소**다.

이유:

* Pod는 생성/삭제되면서 주소가 바뀔 수 있음
* 앞에 고정된 이름이 필요함

비유:

* 직원은 자주 바뀌어도 손님은 대표 전화번호만 알면 되는 구조

핵심:

* Pod 개별 주소를 직접 알 필요 없음
* Service 이름으로 안정적 접근 가능

---

## 14. Helm chart

### 14-1. Helm chart란 무엇인가

Helm chart는 **Kubernetes 배포 서류 묶음 패키지**다.

구성:

```text
templates/
values.yaml
Chart.yaml
```

의미:

* templates = 빈 양식
* values.yaml = 넣을 값
* Chart.yaml = 패키지 정보

핵심:

* 컨테이너를 직접 관리하는 도구가 아님
* 쿠버네티스에 넘길 배포 패키지

---

### 14-2. Helm chart 와 values.yaml 관계

흐름:

```text
values.yaml 값 준비
→ templates 에 값 주입
→ 실제 Kubernetes YAML 생성
```

예시 개념:

```yaml
image:
  tag: a1b2c3d4
```

이 값이 템플릿에 들어가서
실제 Deployment YAML 안의 이미지 버전이 결정된다.

핵심:

* values.yaml 자체가 배포되는 것이 아님
* 템플릿을 채우는 입력값 역할

---

## 15. 이 내용이 소프트웨어 아키텍처와 관련 있는가

관련 있다.
오히려 이번 주 내용은 **소프트웨어 아키텍처의 운영/배포/통합 관점**과 직접 연결된다.

처음에는 단순히 “도구 설명”처럼 보일 수 있지만
실제로는 시스템 구조를 어떻게 나눌지에 대한 아키텍처 선택과 이어진다.

---

### 15-1. 실행 환경 아키텍처

```text
Docker
```

이건 단순 포장이 아니라
**서비스를 환경에 의존하지 않게 배포 가능한 단위로 만드는 아키텍처 선택**이다.

의미:

* 개발 환경 / 운영 환경 차이 축소
* 배포 단위 표준화

---

### 15-2. 배포 아키텍처

```text
Jenkins → GitOps → Argo CD → Kubernetes
```

이건 CI/CD 도구 나열이 아니라
**서비스를 어떻게 안정적으로 릴리스할지에 대한 아키텍처**다.

의미:

* 수동 배포 제거
* 배포 이력 추적
* 운영 자동화

---

### 15-3. 통합 아키텍처

```text
Spring + FastAPI
Spring + Redis
Spring + Kafka
Spring + SQS
```

이건 한 서버 안에 모든 책임을 몰아넣지 않고
**역할별로 시스템을 분리하는 아키텍처**다.

예:

* Spring = 핵심 비즈니스
* FastAPI = 예측 전용 처리
* Redis = 빠른 캐시
* Kafka = 실시간 이벤트 전달
* SQS = 비동기 작업 큐

핵심:

* 역할 분리
* 느슨한 결합
* 확장성 향상

---

### 15-4. 메시징 아키텍처

```text
Kafka / SQS / Pub-Sub
```

이건 단순한 버퍼가 아니라
**서비스 간 결합도를 낮추는 아키텍처 방식**이다.

의미:

* 한 서비스가 다른 서비스를 직접 호출하지 않아도 됨
* 비동기 처리 가능
* 트래픽 급증에도 유리

---

### 15-5. Kubernetes 아키텍처

```text
Pod / Deployment / Service / Namespace
```

이건 단순 용어가 아니라
**서비스를 어떻게 배포 / 복제 / 노출 / 구분할지 정하는 운영 아키텍처**다.

의미:

* 배포 단위 = Pod
* 복제 / 버전 관리 = Deployment
* 통신 창구 = Service
* 환경 구분 = Namespace

---

## 16. 이번 주 핵심 요약

* Dockerfile은 이미지를 만드는 조리법
* Image는 실행 가능한 포장본
* Container는 실제 실행된 상태
* daemon은 뒤에서 계속 동작하는 관리 프로그램
* volume은 컨테이너 바깥 저장공간
* Jenkins는 자동화 서버
* GitOps는 Git을 기준으로 배포 상태를 관리하는 방식
* Argo CD는 Git 상태를 실제 Kubernetes에 반영하는 도구
* values.yaml은 배포할 값 모음
* 이미지 태그는 Docker 이미지 버전표
* MySQL은 DB 엔진
* RDS는 그 DB를 운영해주는 AWS 서비스
* FastAPI는 Python API 서버 프레임워크
* REST는 API 설계 방식
* Redis는 빠른 메모리 저장소
* SQS는 작업 큐
* Kafka는 이벤트 스트림 플랫폼
* Pub/Sub는 느슨하게 연결된 메시징 구조
* Topic은 Kafka 게시판
* Partition은 Topic 안의 병렬 처리 칸
* Offset은 메시지 줄번호
* Cluster는 Kubernetes 전체 공간
* Namespace는 그 안의 구역
* Pod는 가장 작은 배포 단위
* Deployment는 Pod 관리자
* Service는 고정 주소
* Helm chart는 Kubernetes 배포 패키지
* 이번 주 내용은 운영 / 배포 / 통합 / 메시징 관점에서 소프트웨어 아키텍처와 직접 연결됨

---

이번 주는 단순 기술 암기가 아니라
**서비스를 어떻게 포장하고 어떤 환경에서 어떤 방식으로 배포하고, 각 구성요소를 어떤 역할로 나눌지까지 이해한 주차였다.**
