# Week3 — Zone 도메인 클론 코딩 & Spring 구조/문법 정리

## 3주차

아래를 중심으로 정리했다.

- 예외 처리 구조와 역할
- global / dto / util 패키지 의미
- REST API / RESTful 개념
- Controller / Service / RepoService 역할 구분
- Request / Response 기준
- stream / map 동작 원리
- DTO 분리 이유
- IdGenerator 동작 방식
- 트랜잭션 (@Transactional) 차이
- update 로직 흐름 (기존값 / 변경값 / 중복 체크)

---

## 1. 예외 클래스 구조

예외 클래스는 서비스에서 바로 던지기 위해 먼저 정의한다.

```java
throw new NotFoundException("등록된 공간이 없습니다.");
````

핵심:

* 예외는 **에러 상황을 통일해서 표현하는 수단**
* 이후 GlobalExceptionHandler에서 HTTP 응답으로 변환됨

---

## 2. global 패키지 의미

`global`은 특정 도메인에 종속되지 않는 공통 코드 위치다.

사용 예:

* NotFoundException
* BadRequestException
* DuplicateResourceException
* IdGenerator

핵심:

* 여러 도메인에서 재사용되는 코드 → global
* domain 내부에 넣으면 재사용성과 구조가 깨짐

---

## 3. dto 패키지 의미

DTO는 Data Transfer Object

구분:

* Request DTO → 클라이언트 → 서버
* Response DTO → 서버 → 클라이언트

핵심:

* Entity는 내부용 (DB)
* DTO는 외부용 (API)

---

## 4. HTTP 에러 코드 핵심

| 코드  | 의미       |
| --- | -------- |
| 400 | 잘못된 요청   |
| 401 | 인증 없음    |
| 403 | 권한 없음    |
| 404 | 데이터 없음   |
| 409 | 중복/충돌    |
| 500 | 서버 내부 오류 |

핵심:

* 500은 거의 항상 백엔드 문제
* 잘못된 입력은 400으로 처리하는 것이 정상

---

## 5. REST API / RESTful

REST:

* HTTP로 자원을 다루는 방식

RESTful:

* REST 규칙을 잘 지킨 설계

예시:

```text
GET    /zones
POST   /zones
PATCH  /zones/{id}
DELETE /zones/{id}
```

핵심:

* URL = 자원 (명사)
* 행동 = HTTP 메서드

---

## 6. Request / Response 기준

기준은 항상 **클라이언트**

```text
클라이언트 → 요청(Request) → 서버 → 응답(Response) → 클라이언트
```

핵심:

* Request = 클라이언트가 보냄
* Response = 클라이언트가 받음

---

## 7. Controller 역할

Controller는 입출력만 담당한다.

* 요청 받기
* Service 호출
* DTO 반환

핵심:

* 비즈니스 로직은 Service에서 처리

---

## 8. Service 역할

Service는 핵심 로직 담당

흐름:

```text
Controller → Service → RepoService → Repository → DB
```

역할:

* 검증
* 예외 처리
* 비즈니스 규칙 적용
* DTO 변환

---

## 9. RepoService 역할

Repository를 직접 쓰지 않고 한 번 감싸는 계층

역할:

* Optional 처리 통일
* 예외 처리 통일
* 조회/검증 로직 집중

---

## 10. stream / map / collect

조회 코드에서 사용된 핵심 문법

```java
zones.stream()
     .map(ZoneInfoResponse::fromEntity)
     .collect(Collectors.toList());
```

동작:

```text
List<Zone>
→ stream
→ Zone 하나씩 꺼냄
→ DTO로 변환(map)
→ 다시 List로 모음
```

핵심:

* map = 데이터 변환
* collect = 결과 모으기

---

## 11. DTO 분리 이유 (Create / Update)

처음에는 필드가 같아서 하나로 써도 될 것 같았지만, 분리하는 것이 맞다.

이유:

1. 의미가 다름 (생성 vs 수정)
2. 검증 규칙이 달라짐
3. 나중에 필드가 달라질 가능성 높음

핵심:

* 지금이 아니라 **미래를 위해 분리**

---

## 12. Mapper 개념

DTO 변환을 별도 클래스로 분리할 수 있음

```java
ZoneMapper.toResponse(zone);
```

역할:

* Entity → DTO 변환 담당

현재:

* DTO 내부 `fromEntity()` 사용

나중:

* Mapper로 분리 가능

---

## 13. IdGenerator 구조

ID 생성 방식:

```text
현재시간 + "-" + 랜덤 3자리
```

예:

```text
20260327153012-418
```

구성:

* 시간 → 기본 유니크
* 랜덤 → 충돌 방지

---

## 14. Random.nextInt(900) + 100 의미

```java
RANDOM.nextInt(900) + 100
```

범위:

```text
100 ~ 999
```

이유:

* 항상 3자리 숫자 만들기 위해

공식:

```text
nextInt(max - min + 1) + min
```

---

## 15. static factory method

```java
DateTimeFormatter.ofPattern(...)
```

특징:

* 내부에서 객체 생성
* new 안 써도 됨

핵심:

* 메서드가 객체를 대신 생성해줌

---

## 16. @RequiredArgsConstructor

```java
private final ZoneRepoService zoneRepoService;
```

→ 생성자 자동 생성

핵심:

* final 필드만 생성자 포함
* 스프링이 자동 주입(DI)

---

## 17. 트랜잭션 (@Transactional)

기본:

```java
@Transactional
```

→ 읽기 + 쓰기

조회:

```java
@Transactional(readOnly = true)
```

→ 읽기 전용

효과:

* 성능 최적화
* 변경 감지 비활성화

---

## 18. jakarta vs spring Transactional

| 구분       | jakarta | spring |
| -------- | ------- | ------ |
| readOnly | ❌       | ⭕      |
| 확장 기능    | 적음      | 많음     |

핵심:

* Spring Boot에서는 spring.transaction 사용

---

## 19. update 로직 흐름

핵심 구조:

```text
1. 기존 데이터 조회
2. 값 변경 여부 확인
3. 중복 검사
4. 값 변경
5. 저장
```

---

## 20. 기존값 vs 변경값

```text
zoneName → 기존 값 (찾기용)
dto.zoneName → 변경할 값
```

예:

```text
"A존" → "B존"
```

---

## 21. validateZoneName 의미

```java
validateZoneName(dto.getZoneName())
```

역할:

* 새 이름이 DB에 이미 있는지 검사

핵심:

* 중복 방지

---

## 22. update에서 핵심 포인트

잘못된 코드:

```java
zone.setZoneId(...)
```

정답:

```java
zone.setZoneName(...)
```

핵심:

* ID는 절대 변경하지 않음

---

## 23. name으로 조회하는 문제

```java
getZoneByName(zoneName)
```

문제:

* 중복 가능
* 변경 가능

실무:

```java
getZoneById(zoneId)
```

핵심:

* 조회 기준은 항상 유일값

---

## 24. save 동작 방식

```java
save(entity)
```

동작:

* ID 있음 → UPDATE
* ID 없음 → INSERT

핵심:

* 덮어쓰는 것이 아니라 상태 기반 업데이트

---

## 25. 동일 값 업데이트 문제

현재 코드:

```java
if (same) {
    throw exception
}
```

문제:

* 같은 값은 에러가 아니라 “변경 없음”

개선 방향:

* no-op 처리 가능

---

## 26. getAllZones 문제 해결

잘못된 코드:

```java
return findAll().stream()
```

문제:

* DB 두 번 조회

정답:

```java
return zones.stream()
```

핵심:

* 이미 조회한 데이터 재사용

---

## 27. readOnly 에러 원인

```java
import jakarta.transaction.Transactional;
```

→ readOnly 지원 안 함

해결:

```java
import org.springframework.transaction.annotation.Transactional;
```

---

## 28. 전체 흐름 정리

```text
Controller → Service → RepoService → Repository → DB
```

그리고 다시:

```text
Entity → DTO → Controller → 클라이언트
```

---

## 29. 이번 주 핵심 정리

* global은 공통 코드 위치
* DTO는 외부 전달용, Entity는 내부용
* RESTful은 URL=자원, Method=행동
* stream/map은 데이터 변환 흐름
* update는 반드시 기존 조회 → 수정 구조
* 중복 체크는 “새 값” 기준
* ID는 절대 변경하지 않음
* 조회는 반드시 유일값 기준
* save는 ID 기준으로 INSERT/UPDATE 결정
* 트랜잭션은 조회(readOnly) / 수정 구분 필요

---

이번 주는 단순 구현이 아니라
**Spring 구조 + API 설계 + 데이터 흐름을 연결해서 이해한 주차였다.**
`

---

## 30. GET과 POST 차이

HTTP 메서드는 클라이언트가 서버에 어떤 행동을 요청하는지 나타낸다.

구분:

* GET → 데이터 조회
* POST → 데이터 생성 또는 서버 상태 변경

예시:

```text
GET  /api/zones
→ 공간 목록 조회

POST /api/zones
→ 공간 생성
```

핵심:

* GET은 서버 상태를 바꾸지 않는 요청
* POST는 서버 데이터가 바뀌는 요청
* 생성 API를 `@GetMapping`으로 만들면 의미상 잘못된 구조가 됨

---

## 31. Controller에서 잘못 쓰기 쉬운 부분

클론 코딩 중 Controller를 직접 작성하면서 자주 헷갈린 포인트가 있었다.

예시:

```java
@GetMapping
public ZoneInfoResponse create(...)
```

문제:

* 생성 기능인데 GET 사용
* 같은 `@GetMapping`이 두 개면 매핑 충돌 발생

또 다른 예시:

```java
public final ZoneService service;
```

개선:

```java
private final ZoneService service;
```

핵심:

* 생성은 POST, 조회는 GET
* 같은 URL + 같은 HTTP 메서드 조합은 중복되면 안 됨
* 의존성 필드는 `private final`로 두는 것이 기본

---

## 32. Controller에서 RepoService를 직접 호출하면 안 되는 이유

처음에는 Controller에서 `ZoneRepoService`를 바로 호출해도 될 것처럼 보였지만, 계층 구조상 맞지 않았다.

잘못된 흐름:

```text
Controller → RepoService → DB
```

정상 흐름:

```text
Controller → Service → RepoService → Repository → DB
```

핵심:

* Controller는 요청/응답 처리만 담당
* 조회, 검증, 예외 처리 흐름은 Service에서 관리
* Controller가 RepoService를 직접 알기 시작하면 역할 분리가 무너짐

---

## 33. Swagger 역할

Swagger는 API를 문서 형태로 보여주고, 브라우저에서 직접 호출까지 해볼 수 있게 해주는 도구다.

추가 의존성:

```gradle
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8'
```

접속 경로 예시:

```text
http://localhost:8080/swagger-ui.html
```

핵심:

* Postman 없이도 API 테스트 가능
* 어떤 API가 있는지 한눈에 볼 수 있음
* 요청 body와 응답 구조를 문서처럼 확인 가능

---

## 34. Swagger 설정과 설명 추가 위치

Swagger UI를 보기 위해 `application.yml`에 경로 설정을 추가할 수 있다.

예시:

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```

또한 Swagger 설명은 보통 Controller와 DTO에 붙인다.

구분:

* Controller → API 설명
* DTO → 요청/응답 필드 설명

예시:

```java
@Tag(name = "공간 정보 API", description = "공간(Zone) 매핑 처리 API")
@Operation(summary = "공간 생성", description = "공간명을 등록합니다.")
```

핵심:

* API 자체 설명은 Controller에 작성
* 필드 설명은 `@Schema`로 DTO에 작성
* Service나 Repository에는 보통 Swagger 설명을 붙이지 않음

---

## 35. Spring Security 의존성이 있으면 생기는 기본 동작

`spring-boot-starter-security` 의존성이 들어가 있으면, 별도 설정이 없어도 Spring Security가 기본 보안 동작을 활성화한다.

기본적으로 생기는 것:

* 모든 요청에 대한 인증 검사
* CSRF 검사 활성화
* 기본 로그인 동작 적용

그래서 Swagger에서 `POST /api/zones` 호출 시 403이 발생할 수 있었다.

핵심:

* Security 의존성만 있어도 보안 필터가 자동으로 동작
* 초반 CRUD 학습 단계에서는 보안이 오히려 테스트를 방해할 수 있음
* 아직 SecurityConfig를 직접 다루지 않는 단계라면 의존성을 제거하고 기능부터 확인하는 것도 가능함

---

## 36. CSRF 개념

CSRF는 사용자가 의도하지 않은 요청이 브라우저를 통해 서버로 전달되는 공격 방식이다.

상황 예시:

```text
로그인된 사용자가 다른 사이트에 접속
→ 브라우저가 자동으로 인증 정보 포함
→ 원하지 않는 POST 요청 전송 가능
```

Spring Security는 이런 위험한 요청을 막기 위해 POST, PUT, DELETE 같은 요청에 대해 추가 검사를 수행한다.

핵심:

* GET보다 POST 같은 상태 변경 요청에서 문제됨
* Swagger에서 토큰 없이 POST 요청을 보내면 403이 날 수 있음
* Security를 유지한다면 CSRF 설정을 직접 다뤄야 함

---

## 37. SecurityConfig 역할

SecurityConfig는 이 애플리케이션에서 어떤 요청을 허용하고, 어떤 요청을 막을지 정하는 보안 설정 클래스다.

예를 들면:

* 누구나 접근 가능한 URL 설정
* 인증이 필요한 URL 설정
* CSRF 활성화/비활성화
* 로그인 방식 설정

핵심:

* SecurityConfig는 보안 정책을 모아두는 곳
* `permitAll`은 인증 허용이지, CSRF 검사까지 없애는 것은 아님
* 보안 설정이 없으면 기본 보안 규칙이 적용될 수 있음

---

## 38. 이번 단계에서 Security 의존성을 제거한 이유

현재 단계는 Zone CRUD와 Swagger 테스트가 목적이었다.

그래서 아래 의존성을 주석 처리했다.

```gradle
// implementation 'org.springframework.boot:spring-boot-starter-security'
```

핵심:

* 지금은 보안 자체보다 API 구조와 흐름 이해가 더 중요
* Security를 남겨두면 403, CSRF, 인증 설정까지 같이 신경 써야 해서 학습 포인트가 분산됨
* 추후 인증/인가를 배울 때 다시 추가해도 됨

---

## 39. Docker와 MySQL의 관계

이번 프로젝트에서는 MySQL을 로컬에 직접 설치한 것이 아니라 Docker 컨테이너로 실행했다.

전체 흐름:

```text
1. Docker로 MySQL 컨테이너 실행
2. MySQL 서버가 특정 포트에서 대기
3. Spring Boot가 application-local.yml 설정을 읽고 해당 DB에 접속
```

핵심:

* Docker는 DB 서버를 띄우는 역할
* Spring Boot는 그 DB에 접속하는 클라이언트 역할
* Docker가 application-local.yml을 읽는 것이 아니라, Spring이 읽고 접속 정보를 사용함

---

## 40. Docker가 application-local.yml을 읽는 것은 아님

처음에는 Docker로 MySQL을 띄울 때 `application-local.yml`까지 함께 반영된 것처럼 느껴질 수 있지만, 실제로는 다르다.

구분:

* Docker → 컨테이너 실행
* Spring Boot → 설정 파일을 읽고 DB 접속

즉:

```text
Docker는 MySQL 서버만 켠다
Spring은 local.yml을 보고 그 서버에 접속한다
```

핵심:

* Docker와 Spring 설정 파일은 역할이 다름
* 두 설정의 값이 서로 맞아야 연결 성공
* DB 이름, 포트, 비밀번호가 다르면 연결 실패

---

## 41. Docker로 MySQL을 띄우는 기본 흐름

보통 다음 순서로 진행된다.

```text
1. Docker Desktop 실행
2. docker compose up -d
3. MySQL 컨테이너 실행
4. Spring Boot 실행
5. Spring이 DB에 접속
```

확인 명령어:

```bash
docker ps
```

의미:

* 실행 중인 컨테이너 목록 확인
* MySQL 컨테이너가 떠 있어야 Spring Boot가 연결 가능

핵심:

* Docker가 꺼져 있으면 DB 연결 자체가 안 됨
* 이 경우 `Connection refused` 같은 오류가 발생할 수 있음

---

## 42. DB 연결 실패 로그 해석

애플리케이션 실행 중 아래와 같은 에러가 발생했다.

```text
Unable to obtain connection from database
Communications link failure
Connection refused
```

이 의미는:

* 코드 문법 문제는 아님
* Swagger 문제도 아님
* Spring Boot가 DB 서버에 연결하지 못한 상태

핵심:

* Docker로 띄운 MySQL이 꺼져 있거나
* 포트/DB명/비밀번호가 맞지 않거나
* 컨테이너가 정상 실행되지 않은 경우 발생 가능

---

## 43. 이번 단계에서 새로 이해한 전체 실행 흐름

이번에는 단순히 Controller를 작성하는 것을 넘어서, API를 실제로 호출하기 위한 전체 실행 흐름을 함께 이해했다.

흐름:

```text
Docker 실행
→ MySQL 컨테이너 실행
→ Spring Boot 실행
→ application-local.yml 로 DB 접속
→ Flyway 마이그레이션 수행
→ 서버 기동
→ Swagger UI 접속
→ API 테스트
```

핵심:

* API 호출은 Controller 코드만으로 끝나지 않음
* DB, 설정 파일, Docker, Flyway, Swagger가 모두 이어져야 실제 테스트 가능
* 이번 주에는 Spring 코드 구조뿐 아니라 실행 환경까지 함께 연결해서 이해하게 됨

---

## 44. 이번 주 추가 핵심 정리

* GET은 조회, POST는 생성/상태 변경 요청
* Controller는 Service만 호출하고 RepoService를 직접 알지 않는 구조가 맞음
* Swagger는 API 문서화와 테스트 도구
* Swagger 설명은 Controller와 DTO에 붙이는 것이 일반적
* Spring Security 의존성만 있어도 기본 보안 동작이 활성화됨
* CSRF는 상태 변경 요청을 보호하기 위한 보안 검사
* SecurityConfig는 인증/인가/보안 정책을 설정하는 위치
* 현재 단계에서는 Security보다 CRUD와 흐름 이해가 우선이라 의존성을 제거함
* Docker는 DB 서버를 띄우고, Spring Boot는 설정 파일을 읽어 그 DB에 접속함
* 실행 흐름은 코드 작성만이 아니라 Docker, DB, Flyway, Swagger까지 함께 이어져야 함