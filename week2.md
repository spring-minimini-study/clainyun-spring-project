# Week2 — Flyway 초기 스키마 분석 & DB 문법/개념 정리

## 2주차

이번 주에는 `V1__init.sql` 초기 스키마와 Spring/JPA 코드를 같이 보면서,  
**“이 코드/문법이 왜 이렇게 쓰였는지”**를 코드 기준으로 이해하는 데 집중했다.

---

## 1. PK / FK / CONSTRAINT

```sql
CREATE TABLE role_info (
    role_id VARCHAR(50) NOT NULL,
    zone_id VARCHAR(100) NOT NULL,
    PRIMARY KEY (role_id, zone_id),
    CONSTRAINT FK_role_zone
        FOREIGN KEY (zone_id) REFERENCES zone_info (zone_id)
);
````

* `PRIMARY KEY (role_id, zone_id)`

   * `role_id` 하나가 아니라 `role_id + zone_id` 조합으로 한 행을 구분
   * 같은 역할이 여러 공간에 있을 수 있어서 복합키가 필요함
* `FOREIGN KEY (zone_id) REFERENCES zone_info(zone_id)`

   * `role_info.zone_id`는 반드시 `zone_info`에 존재해야 함
* `CONSTRAINT FK_role_zone`

   * 외래키 제약조건에 이름을 붙인 것
   * 기능 자체는 없고, 관리용 이름표 역할
   * 나중에 삭제/수정할 때 유용함

```sql
ALTER TABLE role_info
DROP FOREIGN KEY FK_role_zone;
```

* 외래키 삭제는 **테이블 삭제가 아니라 제약조건만 삭제**
* 삭제 후에는 존재하지 않는 값도 들어갈 수 있음

---

## 2. 복합 외래키

```sql
CREATE TABLE worker_info (
    worker_id VARCHAR(100) NOT NULL,
    role_id VARCHAR(50) NOT NULL,
    zone_id VARCHAR(100) NOT NULL,
    PRIMARY KEY (worker_id),
    CONSTRAINT FK_worker_role
        FOREIGN KEY (role_id, zone_id) REFERENCES role_info (role_id, zone_id)
);
```

* `worker_id`

   * 작업자 자체를 구분하는 PK
* `FOREIGN KEY (role_id, zone_id)`

   * 작업자의 역할-공간 조합이 `role_info`에 실제 존재해야 함
* 왜 `role_id` 하나만 참조 안 하냐

   * `role_info`가 `(role_id, zone_id)` 복합키 구조이기 때문
   * 참조도 같은 기준으로 해야 정확함

---

## 3. COMMENT / TIMESTAMP

```sql
CREATE TABLE zone_hist (
    id BIGINT NOT NULL,
    zone_id VARCHAR(100) NOT NULL,
    worker_id VARCHAR(100) NOT NULL,
    start_time TIMESTAMP NULL DEFAULT NULL,
    end_time TIMESTAMP NULL DEFAULT NULL,
    exist_flag INT COMMENT '1 : 있음\n0 : 떠남',
    PRIMARY KEY (id)
);
```

* `TIMESTAMP`

   * 날짜 + 시간 저장 타입
   * 출입 시각, 감지 시각, 실행 시각 같은 시간 정보에 사용
* `COMMENT`

   * 컬럼 설명용 주석
   * DB 동작에는 영향 없음
* `exist_flag`

   * `1`: 현재 있음
   * `0`: 떠남
* `id`를 PK로 둔 이유

   * 같은 작업자가 같은 공간에 여러 번 출입할 수 있으므로, 기록 자체를 구분해야 함

---

## 4. 로그 테이블에 FK가 없는 이유

```sql
CREATE TABLE abn_log (
    id BIGINT NOT NULL,
    target_type VARCHAR(50),
    target_id VARCHAR(100),
    abnormal_type VARCHAR(100),
    abn_val DOUBLE,
    detected_at TIMESTAMP NULL DEFAULT NULL,
    zone_id VARCHAR(100) NOT NULL,
    PRIMARY KEY (id)
);
```

* 처음에는 `zone_id`에도 FK가 있어야 한다고 생각했음
* 그런데 `abn_log`는 **실시간으로 많이 쌓이는 로그 테이블**
* 이런 테이블은 FK를 걸면:

   * INSERT 때마다 참조 무결성 검사 발생
   * 성능 저하 가능
   * 참조 테이블 문제 시 로그 적재도 같이 막힐 수 있음
* 그래서 로그성 테이블은 **정합성보다 성능/장애 격리**를 우선해서 FK를 생략할 수 있음

핵심:

* 기준 정보 테이블 → FK 적극 사용
* 로그/이력 테이블 → 상황에 따라 FK 생략 가능

---

## 5. abn_log vs notify_log

```sql
CREATE TABLE notify_log (
    abnormal_id BIGINT NOT NULL,
    recipient_id VARCHAR(100) NOT NULL,
    notify_type VARCHAR(50),
    notified_at TIMESTAMP NULL DEFAULT NULL,
    PRIMARY KEY (abnormal_id),
    CONSTRAINT FK_notify_abnormal
        FOREIGN KEY (abnormal_id) REFERENCES abn_log (id)
);
```

* `abn_log`

   * 이상 발생 자체를 기록
   * 원인 로그
* `notify_log`

   * 그 이상 때문에 실제 알림을 보낸 결과를 기록
   * 후속 행동 로그
* 즉

   * `abn_log` = 문제 발생
   * `notify_log` = 문제 때문에 알림 보냄

---

## 6. Kafka

```text
이벤트 발생 → Kafka → 서버 Consumer → 저장 / 알림 / 제어
```

* Kafka는 메시지를 중간에 모아두고 전달하는 시스템
* 역할:

   * 데이터 폭주 시 버퍼 역할
   * 서버가 자기 속도대로 처리 가능
   * 비동기 처리 가능
* 프로젝트에서는 실시간 센서 이벤트를 바로 DB에 꽂기보다,
  Kafka를 통해 한 번 받아서 처리하는 구조로 이해함

---

## 7. Entity

```java
@Entity
@Table(name = "zone_info")
public class Zone {

    @Id
    @Column(name = "zone_id", length = 100, nullable = false, unique = true)
    private String zoneId;

    @Column(name = "zone_name", length = 255, nullable = false)
    private String zoneName;
}
```

* `@Entity`

   * 이 클래스가 JPA 엔티티라는 뜻
   * DB 테이블과 매핑되는 객체
* `@Table(name = "zone_info")`

   * 이 엔티티가 `zone_info` 테이블과 연결됨
* `@Id`

   * PK 필드 지정
* `@Column(...)`

   * 컬럼명, 길이, null 허용 여부 등 설정

핵심:

* Entity = DB 테이블을 Java 객체로 표현한 것

---

## 8. DTO

```java
public class ZoneInfoResponse {
    private String zoneId;
    private String zoneName;

    public static ZoneInfoResponse fromEntity(Zone zone) {
        return new ZoneInfoResponse(zone.getZoneId(), zone.getZoneName());
    }
}
```

* DTO는 데이터 전달용 객체
* Entity를 그대로 외부에 내보내지 않고, 필요한 값만 담아서 전달할 때 사용
* `fromEntity`

   * Entity → DTO 변환 메서드
   * 변환 책임을 DTO 안에 둔 것

핵심:

* Entity = DB용
* DTO = API 응답/요청용

---

## 9. Builder를 안 쓴 이유

```java
return new ZoneInfoResponse(zone.getZoneId(), zone.getZoneName());
```

* 이 DTO는 필드가 2개뿐이라 생성자가 더 간단함
* Builder는 보통:

   * 필드가 많을 때
   * 순서 헷갈릴 때
   * 선택값이 많을 때 사용

핵심:

* DTO라고 무조건 Builder 쓰는 건 아님
* 단순하면 생성자가 더 낫다

---

## 10. Repository

```java
public interface ZoneRepository extends JpaRepository<Zone, String> {
    Optional<Zone> findByZoneName(String zoneName);
    Zone findByZoneId(String zoneId);
}
```

* `JpaRepository<Zone, String>`

   * `Zone` 엔티티를 다루고
   * PK 타입은 `String`
* `findByZoneName(...)`

   * 메서드 이름만으로 쿼리 자동 생성
* `Optional<Zone>`

   * 값이 있을 수도, 없을 수도 있음을 표현

핵심:

* Repository는 DB 직접 접근 창구

---

## 11. RepoService를 따로 둔 이유

```java
@Service
@RequiredArgsConstructor
public class ZoneRepoService {

    private final ZoneRepository zoneRepository;

    public Zone findById(String zoneId) {
        return zoneRepository.findById(zoneId)
                .orElseThrow(() -> new NotFoundException("공간을 찾을 수 없습니다: " + zoneId));
    }
}
```

* `ZoneRepository`를 바로 쓰지 않고 `ZoneRepoService`로 한 번 감쌈
* 이유:

   * Optional 처리 중복 제거
   * 예외 처리 통일
   * 조회/검증 규칙을 한 곳에 모으기 위해
* 구조:

   * Service → RepoService → Repository → DB

핵심:

* 비즈니스 로직(Service)과 DB 접근 규칙(RepoService)을 분리한 구조

---

## 12. @Service / @RequiredArgsConstructor / @Transactional

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class ZoneRepoService {
```

* `@Service`

   * 스프링이 관리하는 서비스 객체로 등록
* `@Slf4j`

   * 로그 객체 자동 생성
* `@RequiredArgsConstructor`

   * `final` 필드를 받는 생성자를 자동 생성

```java
@Transactional
public Zone save(Zone zone) {
    return zoneRepository.save(zone);
}
```

* `@Transactional`

   * DB 작업을 하나의 트랜잭션으로 묶음
   * 저장/수정/삭제에 주로 사용

---

## 13. DI (Dependency Injection)

```java
private final ZoneRepository zoneRepository;
```

* `ZoneRepoService`는 `ZoneRepository` 없이는 동작 못 함
* 즉, `ZoneRepoService`가 `ZoneRepository`에 의존함

```java
@RequiredArgsConstructor
public class ZoneRepoService
```

* 롬복이 생성자를 자동 생성
* 스프링이 이 생성자를 보고 `ZoneRepository`를 넣어줌

핵심:

* DI = 필요한 객체를 내가 직접 `new` 하지 않고, 스프링이 대신 생성해서 넣어주는 것

동작 흐름:

1. 스프링이 Bean 생성
2. `ZoneRepository` Bean 준비
3. `ZoneRepoService` 생성 시 필요한 의존성 확인
4. 생성자로 주입

---

## 14. Bean이 여러 개면?

* 같은 타입 Bean이 여러 개 있으면 스프링이 어떤 걸 넣어야 할지 모름
* 이 경우 에러 발생

해결 방법:

* `@Primary`

   * 기본 Bean 지정
* `@Qualifier`

   * 이름으로 특정 Bean 지정
* `List<T>`

   * 같은 타입 Bean을 전부 주입

---

## 15. Optional / orElseThrow

```java
return zoneRepository.findByZoneName(zoneName)
        .orElseThrow(() -> new ResponseStatusException(
                HttpStatus.NOT_FOUND, "존재하지 않는 공간: " + zoneName));
```

* `findByZoneName(...)`

   * `Optional<Zone>` 반환
* `orElseThrow(...)`

   * 값이 있으면 반환
   * 값이 없으면 예외 발생

핵심:

* null을 직접 다루지 않고, Optional로 안전하게 처리

---

## 16. ResponseStatusException

```java
new ResponseStatusException(HttpStatus.NOT_FOUND, "존재하지 않는 공간: " + zoneName)
```

* HTTP 상태코드와 메시지를 함께 담는 예외
* 예:

   * `HttpStatus.NOT_FOUND` → 404
   * `HttpStatus.BAD_REQUEST` → 400

핵심:

* 단순 Java 예외가 아니라, HTTP 응답까지 의식한 예외 처리

---

## 17. isPresent vs exists

```java
if (zoneRepository.findByZoneName(zoneName).isPresent()) {
    throw new BadRequestException("이미 존재하는 공간명: " + zoneName);
}
```

* `isPresent()`

   * Optional 안에 값이 있는지 확인
   * 먼저 조회를 한 뒤, 값 존재 여부를 판단
* `existsByZoneName(...)`

   * 존재 여부만 바로 확인하는 방식
   * 실제 데이터 전체를 가져오지 않음

핵심:

* 단순 중복 체크는 `existsBy...`가 더 적절함
* 실제 객체가 필요하면 `findBy...` + `orElseThrow` 사용

---

## 18. 예외 종류

* `BadRequestException`

   * 잘못된 요청
   * 보통 400 계열 의미
* `NotFoundException`

   * 데이터 없음
   * 보통 404 계열 의미
* `ResponseStatusException`

   * HTTP 상태 코드와 메시지를 직접 담아 던지는 예외

---

## 19. 아키텍처 관점 정리

### 19.1 계층형 아키텍처

```text
Controller → Service → RepoService → Repository → DB
````

* 역할별로 계층을 나눈 구조
* 각 계층이 자기 책임만 맡도록 분리
* 변경 영향 범위를 줄이기 좋음

---

### 19.2 도메인 중심 패키지 구조

```text
domain
 ├ zone
 ├ sensor
 ├ equip
 ├ worker
```

* 기능별(`controller`, `service`, `repository`)로만 나누는 것이 아니라
* 업무 주제(domain) 기준으로 먼저 묶는 구조
* 관련 코드가 한 곳에 모여서 유지보수하기 좋음

---

### 19.3 현재 프로젝트 구조를 이렇게 이해함

```text
domain/zone
 ├ api
 ├ application
 ├ dao
 ├ dto
 ├ entity
```

* `api` : 외부 요청 받는 계층
* `application` : 서비스/비즈니스 로직 계층
* `dao` : DB 접근 계층
* `dto` : 요청/응답 데이터 전달 객체
* `entity` : DB 테이블 매핑 객체

즉, 이 프로젝트는
**도메인 중심 구조 안에 계층을 함께 나눈 형태**로 이해했다.

---

### 19.4 왜 이렇게 설계하나

* 관련 코드가 흩어지지 않음
* 특정 도메인 수정 시 찾기 쉬움
* 규모가 커질수록 유지보수 유리
* 실무에서 많이 쓰는 방식

---

# 20. 이번주 최종 정리

* PK / FK / 복합키는 테이블 구조를 읽는 핵심 기준이다
* `CONSTRAINT`는 제약조건 이름표다
* 모든 테이블에 FK를 거는 것이 정답은 아니다 -> kafka 실시간성 파괴
* Kafka는 실시간 이벤트를 안정적으로 처리하기 위한 메시징 시스템이다
* Entity와 DTO는 역할이 다르며 분리해야 한다
* Repository를 바로 쓰지 않고 RepoService로 감싸면 구조가 더 정리된다
* 스프링의 DI는 객체 생성을 직접 하지 않게 해준다
* Optional은 null 대신 안전하게 값 유무를 표현한다
* 존재 여부만 확인할 때는 `isPresent`보다 `existsBy...`가 더 적절하다


이번 주는 단순히 SQL 문법을 읽는 수준이 아니라,
**DB 스키마 → JPA 엔티티 → DTO → Repository / Service 구조 → DI → 예외 처리 → Kafka 흐름까지 연결해서 이해한 주차**였다.