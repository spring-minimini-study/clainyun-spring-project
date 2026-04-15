# Week5 — Grafana 연동 구조와 시각화 아키텍처 정리

## 5주차

아래를 중심으로 정리했다.

- Grafana의 개념과 역할
- metrics / logs / traces 의미
- Grafana의 기술 분류와 활용 분야
- 우리 프로젝트에서의 Grafana 연동 구조
- `application.yml` 기반 설정 해석
- iframe / HTTP API / REST API 개념
- Grafana API 호출과 대시보드 생성 흐름
- `DashboardFactory` 역할과 JSON 생성 이유
- datasource, InfluxDB, Prometheus 개념
- Elasticsearch + Kibana 와 Grafana 비교 및 선택 이유
- Grafana 연동 준비 과정
- Grafana 연동의 아키텍처적 의미

---

## 1. Grafana란 무엇인가

Grafana는 **오픈소스 시각화 및 대시보드 플랫폼**이다.

데이터를 직접 저장하거나 처리하는 시스템이라기보다  
**이미 존재하는 데이터를 사람이 보기 쉬운 그래프와 대시보드로 보여주는 도구**에 가깝다.

예를 들어 아래 같은 화면을 만들 때 자주 사용된다.

* 서버 CPU / 메모리 사용량 그래프
* API 응답 시간 그래프
* 공장 센서 온도 / 습도 변화 그래프
* 실시간 운영 현황판

핵심:

* Grafana는 “데이터 저장소”가 아니라 **시각화 계층**
* 숫자와 이벤트를 그래프, 패널, 대시보드 형태로 보여줌

---

## 2. Grafana에서 다루는 데이터

Grafana는 보통 아래 세 가지 데이터를 많이 다룬다.

### 2-1. Metrics

Metrics는 **시간에 따라 계속 쌓이는 숫자 데이터**다.

예:

* CPU 사용률 70%
* API 응답 시간 120ms
* 센서 온도 32도
* 메모리 사용량 2GB

이런 데이터는 시간이 지나면서 계속 바뀌기 때문에 그래프로 보기 좋다.

예시:

```text
10:00  온도 30도
10:01  온도 31도
10:02  온도 29도
````

핵심:

* Metrics는 “시간 기반 숫자 데이터”
* 상태 변화나 추이를 보는 데 적합

---

### 2-2. Logs

Logs는 **문자 형태로 남는 기록 데이터**다.

예:

```text
2026-04-15 10:01:23 INFO Kafka 메시지 수신
2026-04-15 10:01:24 ERROR DB 연결 실패
2026-04-15 10:01:25 WARN 센서 값 이상 감지
```

이건 그래프보다 “무슨 일이 일어났는지”를 확인하는 데 더 적합하다.

핵심:

* Logs는 “사건 기록”
* 에러 원인, 이벤트 발생 시점 추적에 적합

---

### 2-3. Traces

Traces는 **요청 하나가 시스템 안에서 어떤 경로를 거쳤는지 보여주는 데이터**다.

예:

```text
요청
→ Spring 서버
→ DB 조회
→ FastAPI 호출
→ 응답 반환
```

이걸 보면 “어디서 느려졌는지”, “어느 구간에서 병목이 생겼는지”를 추적할 수 있다.

핵심:

* Traces는 “요청 흐름 기록”
* 성능 병목 분석에 적합

---

### 2-4. 정리

| 종류      | 의미                |
| ------- | ----------------- |
| Metrics | 시간에 따라 변하는 숫자 데이터 |
| Logs    | 이벤트 / 에러 기록       |
| Traces  | 요청 흐름 추적          |

---

## 3. Grafana는 어떤 기술인가

Grafana는 아래처럼 이해하는 것이 가장 정확하다.

* AWS 서비스: 아님
* 백엔드 프레임워크: 아님
* Java 라이브러리: 아님
* 데이터베이스: 아님
* **시각화 / 모니터링 / 관측성 플랫폼**: 맞음

즉 **별도로 실행되는 외부 시스템**이고,
백엔드는 그 시스템을 HTTP API와 iframe 방식으로 연동해서 사용한다.

핵심:

* Grafana는 “외부 도구”
* 백엔드가 Grafana를 직접 제어하거나 화면을 가져다 쓰는 구조

---

## 4. Grafana 활용 분야

Grafana는 아래 같은 곳에서 많이 사용된다.

* 인프라 모니터링

    * CPU / 메모리 / 디스크 / 네트워크
* 애플리케이션 모니터링

    * 응답시간 / 에러율 / 요청 수
* IoT / 센서 데이터 시각화

    * 온도 / 습도 / 가스 / 진동
* 운영 대시보드

    * 실시간 현황판 / 상태 모니터링

핵심:

* “운영 상태를 눈으로 바로 확인하는 화면”을 만드는 데 강함

---

## 5. 프로젝트에서 Grafana 역할

우리 프로젝트에서는 아래 흐름으로 이해하면 된다.

```text
센서 데이터 → Grafana → 대시보드 → iframe → 프론트
```

역할:

* 센서 데이터를 그래프로 시각화
* zone별 센서 상태를 화면으로 제공
* 프론트는 Grafana 패널을 iframe으로 삽입해서 사용

핵심:

* 백엔드가 직접 차트를 그리는 구조가 아니라
* Grafana를 시각화 계층으로 두고 연동하는 구조

---

## 6. iframe 개념

iframe은 **웹 페이지 안에 다른 웹 페이지를 끼워 넣는 방식**이다.

예:

```html
<iframe src="https://grafana-url"></iframe>
```

이 의미는
우리 프론트 화면 안에 Grafana 화면 일부를 그대로 넣는다는 뜻이다.

예를 들어 프론트에서 직접 차트를 구현하지 않고
Grafana가 만든 그래프 패널을 그대로 보여줄 수 있다.

핵심:

* iframe = 외부 화면 삽입
* 프론트는 Grafana 화면을 재사용
* 그래프를 따로 다시 만들 필요가 없음

---

## 7. Grafana 설정 (`application.yml`)

```yaml
grafana:
  url:
    outer: ${GRAFANA_URL_OUTER}
    inter: ${GRAFANA_URL_INTER}
  api-key: ${GRAFANA_API_KEY}
  org-id: 1
  datasource-uid: ${GRAFANA_DATASOURCE_UID}
```

이 설정은 단순히 값을 넣어둔 것이 아니라
Grafana를 연동하기 위한 핵심 정보들이다.

### 7-1. outer

* 사용자 화면용 주소
* 브라우저나 iframe에서 접근하는 주소

예:

```text
https://grafana.monitory.space
```

즉 프론트에서 최종적으로 보는 URL이다.

---

### 7-2. inter

* 서버 내부 호출용 주소
* 백엔드가 Grafana API를 호출할 때 쓰는 주소

예:

```text
http://grafana:3000
```

즉 사람이 브라우저에서 보는 주소와
백엔드가 내부적으로 호출하는 주소를 분리한 것이다.

핵심:

* outer = 사람용 주소
* inter = 서버용 주소

---

### 7-3. api-key

* Grafana API 호출 권한
* 백엔드가 “권한 있는 요청”임을 증명하는 값

즉 Grafana API는 아무나 호출하는 게 아니라
API Key를 가진 서버만 호출할 수 있도록 막아두는 경우가 많다.

---

### 7-4. org-id

* Grafana 내부 조직 구분 값
* 여러 팀이나 환경을 나눠 쓰는 경우 사용

예:

* org 1 = 개발팀
* org 2 = 운영팀

우리 프로젝트에서는 기본값 1을 사용한다.

---

### 7-5. datasource-uid

* Grafana가 데이터를 가져올 데이터 소스 식별자

예:

* InfluxDB
* Prometheus
* MySQL

Grafana는 데이터를 직접 저장하지 않기 때문에
“어디서 데이터를 읽을지”를 알아야 한다.
그 역할을 하는 것이 datasource 설정이다.

핵심:

* datasource = 데이터 읽는 위치
* datasource-uid = 그 위치를 가리키는 식별자

---

## 8. InfluxDB란 무엇인가

InfluxDB는 **시계열(Time-series) 데이터베이스**다.

즉 “시간에 따라 계속 쌓이는 데이터”를 저장하는 데 특화된 DB다.

예:

```text
시간    센서온도
10:00   30
10:01   31
10:02   29
```

이런 데이터는 일반 DB에도 저장할 수는 있지만,
시간 기준 집계나 최근 구간 조회가 많아질수록 시계열 DB가 훨씬 유리하다.

핵심:

* InfluxDB = 시간 기반 데이터 저장용 DB
* 센서 데이터처럼 “계속 쌓이는 숫자 데이터”에 적합

---

## 9. 왜 굳이 InfluxDB가 필요한가

센서 데이터는 아래처럼 계속 발생한다.

```text
1초마다 100개 센서 데이터 발생
→ 1분이면 6,000건
→ 1시간이면 360,000건
```

이걸 일반 DB에 저장하면 조회와 집계가 점점 무거워질 수 있다.

Grafana에서 자주 하는 작업:

* 최근 15분 평균값
* 시간별 최대값
* 센서별 추이 그래프

이런 작업은 **시간 기준 조회와 집계**가 핵심인데,
InfluxDB는 이런 용도에 맞게 설계되어 있다.

핵심:

* 일반 DB도 저장은 가능
* 하지만 시간 기반 집계 / 추이 조회는 InfluxDB가 더 적합
* 센서 데이터, 메트릭 데이터와 궁합이 좋음

---

## 10. Prometheus란 무엇인가

Prometheus는 **메트릭 수집 + 저장 시스템**이다.

InfluxDB가 “저장소” 느낌이라면
Prometheus는 **메트릭을 자동으로 수집해오는 도구**에 더 가깝다.

예를 들어 서버가 아래 같은 상태 값을 노출한다고 하자.

```text
CPU 70%
메모리 80%
요청 수 1200
```

Prometheus는 이런 값을 일정 주기로 자동 수집해서 저장한다.

핵심:

* Prometheus = 메트릭 수집기 + 시계열 저장소
* 서버 / 인프라 상태 모니터링에 매우 많이 사용됨

---

## 11. 왜 Prometheus가 필요한가

**있으면 훨씬 편하다**!!

### 11-1. Prometheus가 없으면

서버 상태 데이터를 직접 수집해서 DB에 넣어야 한다.

예:

```text
애플리케이션 코드
→ CPU 값 읽기
→ 메모리 값 읽기
→ DB 저장
→ 그래프 그리기
```

즉 수집 로직을 전부 직접 구현해야 한다.

---

### 11-2. Prometheus가 있으면

Prometheus가 자동으로 메트릭을 긁어간다.

```text
Prometheus
→ /metrics 엔드포인트 접근
→ 값 수집
→ 저장
```

그래서 개발자는 수집기를 직접 만들지 않아도 된다.

---

### 11-3. Prometheus가 특히 좋은 경우

* 서버 상태를 자동 수집하고 싶을 때
* CPU / 메모리 / 요청 수 같은 운영 지표를 보고 싶을 때
* 임계치 초과 시 알림까지 연결하고 싶을 때

예:

```text
CPU 90% 초과 → Alert
```

핵심:

* Prometheus는 “수집 자동화” 때문에 필요
* 없어도 구현은 가능하지만 직접 다 만들어야 함
* 모니터링 시스템 구축에는 매우 유리함

---

## 12. InfluxDB와 Prometheus 차이

둘 다 Grafana와 자주 같이 언급되지만 역할이 다르다.

| 구분    | InfluxDB        | Prometheus        |
| ----- | --------------- | ----------------- |
| 핵심 역할 | 시간 데이터 저장       | 메트릭 자동 수집 + 저장    |
| 강점    | 센서 / IoT 데이터 저장 | 서버 / 인프라 메트릭 수집   |
| 방식    | 보통 애플리케이션이 넣어줌  | Prometheus가 직접 수집 |
| 성격    | 저장소             | 수집기 + 저장소         |


---

## 13. 프로젝트에서 왜 둘 다 같이 쓰는가


```text
센서 데이터 → InfluxDB
서버 상태 → Prometheus
Grafana → 둘 다 읽어서 그래프 생성
```

즉

* 센서 데이터처럼 애플리케이션이 직접 발생시키는 데이터는 InfluxDB에 저장
* 서버 상태처럼 자동 모니터링이 필요한 메트릭은 Prometheus가 수집
* Grafana는 이 둘을 가져와서 한 화면에 보여줌

핵심:

* InfluxDB와 Prometheus는 **역할이 다른 데이터 소스**
* Grafana는 여러 데이터소스를 한 화면에 묶어서 보여줄 수 있음

---

## 14. Grafana 연동 준비 과정

Grafana를 붙이려면 단순히 주소만 있으면 되는 게 아니다.
보통 아래 과정을 거친다.

### 14-1. Grafana 서버 실행

먼저 Grafana 자체가 실행 중이어야 한다.

예:

```text
Grafana 서버 기동
```

---

### 14-2. 데이터소스 연결

Grafana UI에서 어떤 데이터소스를 읽을지 등록한다.

예:

```text
Settings → Data Sources → Add data source
```

이때 InfluxDB나 Prometheus 같은 데이터 소스를 연결한다.

---

### 14-3. API Key 발급

Grafana API를 백엔드에서 호출하려면 인증 키가 필요하다.

보통 Grafana UI에서 API Key를 발급하고
그 값을 `application.yml` 혹은 환경변수로 넣는다.

즉:

* API Key 발급
* 백엔드 설정에 주입
* `BearerAuth`로 API 호출

---

### 14-4. datasource UID 확인

Grafana는 데이터소스를 이름만이 아니라 UID로 식별하는 경우가 많다.

그래서 백엔드가 JSON으로 대시보드를 생성할 때
어떤 데이터소스를 연결할지 UID를 알아야 한다.

핵심:

* Grafana 연동 준비 = 서버 실행 + 데이터소스 등록 + API Key 발급 + UID 확인

---

## 15. GrafanaClient (API 호출)

```java
headers.setBearerAuth(apiKey);

restTemplate.exchange(
    "/api/dashboards/db",
    HttpMethod.POST,
    entity,
    ...
);
```

이 코드는 백엔드가 Grafana에게 직접 HTTP 요청을 보내는 부분이다.

흐름:

* API Key로 인증
* `/api/dashboards/db` 호출
* 대시보드 생성 요청
* 결과 응답 수신

핵심:

* 단순 링크 연결이 아니라 **백엔드가 Grafana를 직접 제어**
* 이 구조 덕분에 대시보드를 수동 생성하지 않아도 됨

---

## 16. HTTP API와 REST API 관계

```text
HTTP API ⊃ REST API
```

### HTTP API

* HTTP 프로토콜을 사용하는 모든 API

예:

```text
GET /api/users
POST /login
```

---

### REST API

* HTTP API 중에서 REST 규칙을 따르는 API

규칙 예:

* URL = 자원
* Method = 행위
* Stateless

예:

```text
GET /users
POST /users
DELETE /users/{id}
```

---

### Grafana API는 무엇인가

```java
restTemplate.exchange(
    "/api/dashboards/db",
    HttpMethod.POST,
    ...
);
```

이건 HTTP API 호출이고,
형태상 대부분 REST 스타일로 이해하면 된다.

핵심:

* Grafana와의 통신은 HTTP 기반
* 대체로 REST 스타일 엔드포인트 호출 구조

---

## 17. UID 반환

Grafana는 대시보드를 생성하면
그 대시보드를 구분할 수 있는 **고유 식별자(uid)** 를 반환한다.

예:

```text
abc123xyz
```

이건 말 그대로

```text
“방금 만든 대시보드가 누구인지 구분하는 ID”
```

라는 뜻이다.

이 UID가 있으면 이후에:

* 대시보드 다시 찾기
* URL 만들기
* 특정 대시보드 패널 참조

같은 작업을 할 수 있다.

핵심:

* UID = 대시보드 주민등록번호 같은 것
* 생성 후 다시 접근하거나 URL 조합할 때 필요

---

## 18. DashboardFactory란 무엇인가

Factory는 보통 **객체를 생성하는 역할을 전담하는 클래스**를 뜻한다.

즉 여기서는

```text
DashboardFactory = Grafana 대시보드 정의를 만들어주는 전용 클래스
```

라고 이해하면 된다.

핵심:

* 대시보드 생성 로직을 한곳에 모음
* Service에서 직접 JSON 조립하지 않게 역할 분리

---

## 19. JSON을 왜 생성하는가

Grafana는 대시보드를 내부적으로 JSON 형태로 정의한다.

예를 들어 아래처럼 “대시보드 설계도”가 JSON으로 표현된다.

```json
{
  "title": "zone1",
  "refresh": "2s",
  "panels": [...]
}
```

즉 JSON은 단순 문자열이 아니라
**어떤 패널을 어떤 이름으로 어떤 데이터소스에서 어떤 시간 범위로 보여줄지 적어놓은 설계도**다.

그래서 백엔드가 해야 하는 일은:

```text
zone 정보 / 센서 정보 조회
→ Grafana 대시보드 설계도(JSON) 생성
→ Grafana API에 전달
```

이 된다.

핵심:

* JSON = 대시보드 설계도
* Grafana는 그 설계도를 받아 실제 대시보드 생성

---

## 20. DashboardFactory 코드 의미

```java
dash.put("title", zoneId);
dash.put("refresh", "2s");

tgt.put("measurement", "ENVIRONMENT");

tagFilter.put("value", sensorId);
```

이 코드는 아래를 의미한다.

* `title` → 대시보드 제목
* `refresh` → 몇 초마다 새로고침할지
* `measurement` → 어떤 데이터 계열을 볼지
* `sensorId` 필터 → 어떤 센서 데이터만 볼지

즉 이 코드는
대시보드의 모양만 만드는 게 아니라
**무슨 데이터를 어떤 조건으로 어떤 이름으로 보여줄지 정하는 코드**다.

---

## 21. GrafanaZoneService 흐름

```text
센서 조회
→ JSON 생성
→ Grafana API 호출
→ dashboard UID 생성
→ iframe URL 생성
```

이 흐름이 중요한 이유는
우리 프로젝트의 Grafana 연동이
**“조회 → 생성 → 연결” 전체를 백엔드가 자동화하고 있다는 점**을 보여주기 때문이다.

---

## 22. iframe URL 구조

```java
"%s/d-solo/%s/%s?orgId=%d&panelId=%d"
```

이 URL은 “Grafana 대시보드 전체”가 아니라
**특정 패널 하나만 잘라서 보여주는 주소**에 가깝다.

즉:

* 특정 대시보드
* 특정 패널
* 특정 조직
* 특정 시간 범위

를 조합해서 프론트에 넘기는 것이다.

핵심:

* 전체 화면이 아니라 필요한 패널만 프론트에 삽입 가능

---

## 23. Elasticsearch와의 차이

| 구분 | Grafana    | Elasticsearch |
| -- | ---------- | ------------- |
| 역할 | 시각화        | 저장 / 검색       |
| 강점 | 그래프 / 대시보드 | 검색 / 분석       |

핵심:

* Grafana = 보여주는 도구
* Elasticsearch = 저장하고 찾는 엔진

---

## 24. Kibana와 비교

| 도구            | 역할                      |
| ------------- | ----------------------- |
| Elasticsearch | 데이터 엔진                  |
| Kibana        | Elasticsearch 전용 시각화 UI |
| Grafana       | 범용 시각화 플랫폼              |

즉:

* Elasticsearch + Kibana 는 한 세트처럼 많이 사용됨
* Grafana는 여러 종류의 데이터소스를 더 범용적으로 붙일 수 있음

---

## 25. 왜 Elasticsearch + Kibana 와 Grafana를 같이 고민했는가

처음 보면 둘 다 “데이터를 보여주는 도구”처럼 느껴진다.

예:

* 둘 다 그래프를 띄울 수 있음
* 둘 다 대시보드를 만들 수 있음
* 둘 다 운영 화면처럼 쓸 수 있음

그래서 처음 도입 시점에는
“둘 중 뭘 써야 하지?”라는 고민이 자연스럽게 생긴다.

---

## 26. 같은 기능을 할 수 있는가

완전히 같지는 않지만, **겹치는 영역은 있다.**

예를 들어

* Grafana도 그래프를 그릴 수 있음
* Kibana도 시각화가 가능함

그래서 “대시보드 화면”이라는 결과만 보면 비슷해 보일 수 있다.

하지만 핵심 목적은 다르다.

### Elasticsearch + Kibana

* 로그 검색, 문서 검색, 텍스트 분석에 강함

### Grafana

* 시간 기반 숫자 데이터 시각화, 실시간 모니터링에 강함

핵심:

* 일부 기능은 겹쳐 보이지만
* 강한 영역이 다름

---

## 27. 왜 우리 프로젝트는 Grafana를 선택했는가

우리 프로젝트 데이터의 성격은 아래와 같았다.

* 센서 데이터
* 시간 흐름에 따라 계속 쌓임
* 숫자 기반
* 실시간 모니터링 중요

즉 핵심은

```text
“로그 검색”보다 “시간 기반 메트릭 시각화”
```

였다.

이 기준으로 보면:

### Elasticsearch + Kibana

* 로그 분석에는 강함
* 검색 / 필터링 중심
* 텍스트 데이터와 잘 맞음

### Grafana

* 시간 기반 수치 데이터에 강함
* 실시간 그래프에 적합
* InfluxDB / Prometheus 와 자연스럽게 연결 가능

그래서 우리 프로젝트에서는
**센서 기반 실시간 모니터링이라는 목적에 더 잘 맞는 Grafana를 선택한 것**으로 정리할 수 있다.

핵심:

* Elasticsearch + Kibana = 로그 분석 쪽에 더 강함
* Grafana = 실시간 메트릭 시각화에 더 적합
* 우리 프로젝트는 후자에 가까움

---

## 28. 아키텍처 관점 의미

### 28-1. 시각화 계층 분리

```text
Backend ≠ 화면
Grafana = 시각화 담당
```

즉 백엔드가 모든 화면을 직접 책임지지 않고
시각화 역할을 Grafana에 분리했다는 의미다.

---

### 28-2. 외부 시스템 연동

* Grafana는 외부 시스템
* 백엔드는 HTTP API로 제어
* iframe으로 화면을 끼워 넣음

즉 “외부 관측성 시스템과의 연동 구조”라고 볼 수 있다.

---

### 28-3. 자동화

* zone별 대시보드 자동 생성
* 센서별 URL 자동 생성

즉 운영자가 수동으로 화면을 만들지 않아도 되는 구조다.

---

### 28-4. 역할 분리

```text
Spring → 비즈니스 로직
Grafana → 시각화
InfluxDB / Prometheus → 데이터 소스
```

이렇게 역할을 나누면 구조가 더 명확해진다.

---

## 29. 전체 구조

```text
Controller
→ Service
   → 센서 조회
   → DashboardFactory로 JSON 생성
   → GrafanaClient로 API 호출
   → UID 수신
   → iframe URL 생성
→ 반환
→ 프론트가 iframe으로 Grafana 패널 표시
```

핵심:

* Controller = 입출력
* Service = 전체 흐름 조합
* Factory = 대시보드 설계도 생성
* Client = 외부 Grafana API 호출

---

## 30. 핵심 요약

* Grafana는 시각화 / 관측성 플랫폼이다
* 외부 시스템이며 백엔드는 API와 iframe으로 연동한다
* metrics는 시간 기반 숫자 데이터다
* InfluxDB는 센서 같은 시계열 데이터를 저장하기 위해 사용한다
* Prometheus는 서버 메트릭을 자동 수집하기 위해 사용한다
* JSON은 Grafana 대시보드 설계도 역할을 한다
* DashboardFactory는 그 설계도를 만들어주는 전용 클래스다
* Grafana는 대시보드를 만들면 UID를 반환하고, 그 UID로 다시 접근할 수 있다
* Elasticsearch + Kibana 와 Grafana는 일부 결과가 비슷해 보여도 강점이 다르다
* 우리 프로젝트는 로그 검색보다 실시간 센서 모니터링이 중요했기 때문에 Grafana가 더 적합했다

---

## 31. 한 줄 정리

Grafana는 시간 기반 데이터와 운영 상태를 대시보드로 시각화하는 외부 플랫폼이며,
우리 프로젝트에서는 InfluxDB / Prometheus 같은 데이터 소스와 연결하고,
백엔드가 API를 통해 대시보드를 자동 생성한 뒤 iframe 방식으로 프론트에 제공하는 구조로 연동되어 있다.
