# Week8 — Redis TTL, 롱폴링, FastAPI 연동 구조 정리

## 8주차

이번 주에는 백엔드에서 자주 등장하는 세 가지 구조를 실제 프로젝트 코드 기준으로 정리했다.

첫 번째는 Redis TTL이다. 인증번호, 리프레시 토큰, 임시 사용자 정보처럼 일정 시간이 지나면 자동으로 사라져야 하는 데이터를 Redis에서 어떻게 관리하는지 봤다.

두 번째는 롱폴링이다. WebSocket을 쓰지 않고도 채팅에서 준실시간 응답을 만드는 방식을 코드 흐름으로 이해했다.

세 번째는 FastAPI 연동이다. Spring 서버가 S3 업로드 이벤트를 SQS로 감지하고, 그 흐름에서 FastAPI 예측 API를 호출한 뒤 결과를 서비스 로직으로 연결하는 구조를 정리했다.

이번 주에 정리한 핵심 내용은 다음과 같다.

- Redis TTL이 필요한 이유
- @RedisHash, @TimeToLive, @Indexed 의미
- 인증번호 TTL 흐름
- RefreshToken TTL 흐름
- OAuth 가입 중 임시 사용자 정보 TTL 흐름
- 롱폴링이 필요한 이유
- short polling, long polling, WebSocket 차이
- lastMessageId를 기준으로 신규 메시지를 조회하는 방식
- 롱폴링의 장점과 한계
- FastAPI를 Spring에서 호출하는 방식
- RestTemplate을 Bean으로 등록하는 이유
- S3 이벤트 → SQS → Spring Listener → FastAPI 호출 흐름
- UriComponentsBuilder, @Value, ResponseEntity, DTO 역직렬화 의미
- 예측 결과를 DB 저장, Slack 알림, 이상 로그 저장으로 연결하는 방식

---

# 1. Redis TTL

## 1.1 Redis란 무엇인가

Redis는 메모리 기반의 데이터 저장소다.

일반적인 MySQL이나 PostgreSQL 같은 RDB는 데이터를 디스크 중심으로 저장한다. 반면 Redis는 메모리를 중심으로 데이터를 저장하기 때문에 읽고 쓰는 속도가 빠르다.

그래서 Redis는 다음과 같은 데이터에 자주 사용된다.

- 인증번호
- 로그인 세션
- 리프레시 토큰
- 임시 사용자 정보
- 캐시 데이터
- 중복 요청 방지용 값
- 짧은 시간만 필요한 상태값

이런 데이터들은 영구적으로 저장할 필요가 없다. 일정 시간이 지나면 자동으로 사라지는 것이 더 자연스럽다.

예를 들어 휴대폰 인증번호가 5분 뒤에도 계속 살아 있으면 보안상 위험하다. 리프레시 토큰도 만료 시간이 지났는데 Redis에 남아 있으면 관리가 꼬일 수 있다.

이때 사용하는 개념이 TTL이다.

---

## 1.2 TTL이란 무엇인가

TTL은 Time To Live의 줄임말이다.

의미는 다음과 같다.

```text
데이터가 살아 있을 수 있는 시간
````

예를 들어 TTL이 300초라면, Redis에 저장된 데이터는 300초 뒤 자동으로 삭제된다.

```text
Redis에 저장
↓
300초 동안 유지
↓
300초 뒤 자동 삭제
```

이 구조를 쓰면 개발자가 직접 만료 시간을 검사하고 삭제 배치를 돌리지 않아도 된다.

---

## 1.3 Redis TTL을 쓰는 이유

Redis TTL을 쓰는 이유는 명확하다.

일정 시간이 지나면 의미가 없어지는 데이터가 있기 때문이다.

예를 들어 다음 데이터들이 그렇다.

| 데이터          | 왜 TTL이 필요한가                 |
| ------------ | --------------------------- |
| 휴대폰 인증번호     | 5분이 지나면 더 이상 유효하면 안 됨       |
| RefreshToken | JWT 만료시간과 Redis 저장시간이 맞아야 함 |
| 임시 사용자 정보    | 회원가입 중단 시 계속 남아 있으면 안 됨     |

즉 Redis TTL은 단순한 편의 기능이 아니라 보안과 데이터 정합성을 위해 사용된다.

---

# 2. Redis TTL 핵심 어노테이션

## 2.1 @RedisHash

```java
@RedisHash(value = "certification_code")
public class CertificationCode {
}
```

@RedisHash는 이 객체를 Redis에 저장할 엔티티로 보겠다는 의미다.

JPA에서 @Entity가 DB 테이블과 매핑되는 것처럼, Redis에서는 @RedisHash가 Redis 저장 구조와 연결된다.

```text
JPA Entity → RDB 테이블에 저장
RedisHash → Redis에 저장
```

value 값은 Redis에서 key prefix처럼 사용된다.

예를 들어 다음과 같이 설정하면:

```java
@RedisHash(value = "certification_code")
```

Redis에는 대략 다음 느낌으로 저장된다.

```text
certification_code:01012345678
```

---

## 2.2 @Id

```java
@Id
private String phone;
```

Redis에서 해당 데이터를 구분하는 key 역할을 한다.

예를 들어 phone이 01012345678이면, 이 전화번호를 기준으로 인증번호 데이터를 찾을 수 있다.

CertificationCode에서는 전화번호가 key다.

```text
전화번호 하나당 인증번호 하나
```

같은 번호로 인증번호를 다시 요청하면 기존 값이 덮어씌워질 수 있다.

---

## 2.3 @TimeToLive

```java
@TimeToLive
private Long expiresIn;
```

@TimeToLive는 이 필드 값을 TTL로 사용하겠다는 의미다.

단위는 보통 초다.

예를 들어 expiresIn이 300이면 Redis에 저장된 뒤 300초 후 자동으로 삭제된다.

```text
expiresIn = 300
→ Redis 저장 후 5분 뒤 자동 삭제
```

이게 Redis TTL의 핵심이다.

---

## 2.4 @Indexed

```java
@Indexed
private String accessToken;
```

@Indexed는 해당 필드로 조회할 수 있게 인덱스를 걸어주는 역할을 한다.

Redis는 기본적으로 @Id 기준 조회가 가장 자연스럽다. 그런데 accessToken 같은 다른 필드로도 찾아야 할 때가 있다.

그럴 때 @Indexed를 붙여 조회 가능성을 열어둔다.

---

# 3. Redis TTL 적용 구조 1 — 휴대폰 인증 코드

## 3.1 목적

휴대폰 인증 코드는 짧은 시간만 유효해야 한다.

예를 들어 사용자가 휴대폰 인증번호를 요청하면 6자리 코드가 발급된다.

```text
사용자 휴대폰 인증 요청
→ 인증번호 생성
→ Redis에 5분 TTL로 저장
→ 사용자가 인증번호 입력
→ 인증 성공 시 즉시 삭제
→ 5분이 지나면 자동 삭제
```

인증번호가 5분 뒤에도 계속 살아 있으면 다른 사람이 나중에 악용할 수 있다.

그래서 인증번호는 Redis TTL과 잘 맞는다.

---

## 3.2 application-dev.yml 설정

```yaml
verify-phone:
    expires-in: ${VERIFY_PHONE_EXPIRES_IN:300}
```

이 설정은 휴대폰 인증번호 만료 시간을 의미한다.

```text
VERIFY_PHONE_EXPIRES_IN 환경변수가 있으면 그 값을 사용
없으면 기본값 300초 사용
```

300초는 5분이다.

즉 기본적으로 인증번호는 5분 동안만 유효하다.

---

## 3.3 CertificationCode Redis 엔티티

```java
package com.caring.caringbackend.domain.auth.entity;

import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.redis.core.RedisHash;
import org.springframework.data.redis.core.TimeToLive;

import java.time.LocalDate;

@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@RedisHash(value = "certification_code")
public class CertificationCode {

    @Id
    private String phone;

    private String code;

    private String name;

    private LocalDate birthDate;

    @TimeToLive
    private Long expiresIn;
}
```

---

## 3.4 코드 설명

```java
@RedisHash(value = "certification_code")
```

이 클래스는 Redis에 저장되는 객체다.

---

```java
@Id
private String phone;
```

전화번호가 Redis key 역할을 한다.

즉 같은 전화번호로 인증번호를 다시 요청하면 같은 key에 저장되므로 기존 값이 갱신될 수 있다.

---

```java
private String code;
```

실제 인증번호다.

예를 들어 123456 같은 값이 들어간다.

---

```java
private String name;
private LocalDate birthDate;
```

인증 과정에서 함께 검증하거나 이후 가입 흐름에서 사용할 수 있는 사용자 정보다.

---

```java
@TimeToLive
private Long expiresIn;
```

TTL 값이다.

이 값이 300이면 Redis 저장 후 300초 뒤 자동 삭제된다.

---

## 3.5 인증번호 저장 코드

```java
CertificationCode certificationCode = CertificationCode.builder()
        .code(code)
        .phone(phoneNumber)
        .name(certificationCodeRequest.getName())
        .birthDate(certificationCodeRequest.getBirthDate())
        .expiresIn(certificationCodeExpiresIn)
        .build();

certificationCodeRepository.save(certificationCode);
```

---

## 3.6 저장 흐름

```text
1. 사용자가 휴대폰 인증 요청

2. 서버가 인증번호 생성

3. CertificationCode 객체 생성

4. expiresIn에 yml에서 가져온 300초 설정

5. Redis에 저장

6. 300초 뒤 Redis에서 자동 삭제
```

---

## 3.7 인증 성공 시 즉시 삭제

```java
public void verifyPhone(VerifyPhoneRequest verifyPhoneRequest) {
    CertificationCode certificationCode = certificationCodeRepository
            .findCertificationCodeByPhone(verifyPhoneRequest.getPhoneNumber())
            .orElseThrow(() -> new BusinessException(
                    ErrorCode.VALIDATION_ERROR,
                    "인증번호가 만료되었습니다."
            ));

    if (!certificationCode.getCode().equals(verifyPhoneRequest.getCode())) {
        throw new BusinessException(
                ErrorCode.VALIDATION_ERROR,
                "인증번호가 올바르지 않습니다."
        );
    }

    transactionTemplate.executeWithoutResult(
            status -> certificationCodeRepository.delete(certificationCode)
    );
}
```

---

## 3.8 코드 설명

```java
findCertificationCodeByPhone(...)
```

전화번호로 Redis에서 인증번호를 찾는다.

만약 TTL이 만료됐다면 Redis에서 이미 삭제되었으므로 조회되지 않는다.

그래서 이 경우 인증번호가 만료되었다는 예외를 던진다.

---

```java
if (!certificationCode.getCode().equals(verifyPhoneRequest.getCode()))
```

사용자가 입력한 인증번호와 Redis에 저장된 인증번호를 비교한다.

다르면 인증 실패다.

---

```java
certificationCodeRepository.delete(certificationCode)
```

인증이 성공하면 TTL 만료를 기다리지 않고 즉시 삭제한다.

이유는 인증이 끝난 코드는 더 이상 필요 없기 때문이다.

인증번호는 한 번 쓰고 버리는 값에 가깝다.

---

## 3.9 휴대폰 인증 코드 흐름 정리

```text
인증번호 요청
→ Redis에 CertificationCode 저장
→ TTL 300초 설정
→ 사용자가 인증번호 입력
→ Redis에서 전화번호로 조회
→ 코드 비교
→ 성공하면 즉시 삭제
→ 실패하거나 만료되면 예외
```

---

# 4. Redis TTL 적용 구조 2 — RefreshToken

## 4.1 목적

RefreshToken은 AccessToken이 만료되었을 때 새로운 AccessToken을 발급받기 위해 사용하는 토큰이다.

하지만 RefreshToken도 영원히 유효하면 안 된다.

JWT 자체에도 만료시간이 있고, Redis에서도 같은 만료시간을 갖도록 저장한다.

이렇게 하면 다음 효과가 있다.

```text
JWT 만료
= Redis에서도 자동 삭제
```

또한 로그아웃 시 Redis에서 RefreshToken을 삭제하면, 아직 JWT 만료시간이 남아 있어도 재발급을 막을 수 있다.

---

## 4.2 RefreshToken Redis 엔티티

```java
package com.caring.caringbackend.domain.auth.entity;

import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.redis.core.Indexed;
import org.springframework.data.redis.core.RedisHash;
import org.springframework.data.redis.core.TimeToLive;

@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@RedisHash(value = "refresh_token")
public class RefreshToken {

    @Indexed
    @Id
    private String refreshToken;

    @TimeToLive
    private Long expiresIn;
}
```

---

## 4.3 코드 설명

```java
@RedisHash(value = "refresh_token")
```

RefreshToken을 Redis에 저장한다.

---

```java
@Id
private String refreshToken;
```

JWT 리프레시 토큰 문자열 자체가 key다.

즉 Redis에 저장된 토큰인지 확인할 수 있다.

---

```java
@TimeToLive
private Long expiresIn;
```

RefreshToken의 Redis TTL이다.

JWT refresh token validity 값과 맞춰 저장한다.

---

## 4.4 RefreshToken 저장 코드

```java
private void saveRefreshToken(String refreshToken, Long expiresIn) {
    RefreshToken token = RefreshToken.builder()
            .refreshToken(refreshToken)
            .expiresIn(expiresIn)
            .build();

    refreshTokenRepository.save(token);
}
```

---

## 4.5 저장 흐름

```text
1. 로그인 성공

2. AccessToken, RefreshToken 생성

3. RefreshToken을 Redis에 저장

4. expiresIn에 JWT RefreshToken 유효시간 설정

5. 토큰 만료 시 Redis에서도 자동 삭제
```

---

## 4.6 토큰 재발급 시 검증 코드

```java
public RefreshTokenPayloadDto decodeRefreshToken(TokenRefreshRequest tokenRefreshRequest) {
    String refreshToken = tokenRefreshRequest.getRequestToken();

    if (jwtUtils.isTokenExpired(refreshToken)) {
        throw new BusinessException(ErrorCode.TOKEN_EXPIRED);
    }

    if (refreshTokenRepository.findRefreshTokenByRefreshToken(refreshToken).isEmpty()) {
        throw new BusinessException(ErrorCode.TOKEN_EXPIRED);
    }

    return jwtUtils.decodeRefreshToken(refreshToken);
}
```

---

## 4.7 코드 설명

```java
jwtUtils.isTokenExpired(refreshToken)
```

1차로 JWT 자체가 만료되었는지 확인한다.

JWT의 서명과 만료시간을 보는 과정이다.

---

```java
refreshTokenRepository.findRefreshTokenByRefreshToken(refreshToken).isEmpty()
```

2차로 Redis에 해당 RefreshToken이 존재하는지 확인한다.

JWT 자체가 아직 만료되지 않았더라도 Redis에 없으면 재발급을 막는다.

Redis에 없는 경우는 보통 다음과 같다.

* TTL이 지나 자동 삭제됨
* 로그아웃하면서 삭제됨
* 서버에서 강제로 무효화함
* 탈취된 토큰일 가능성이 있음

---

## 4.8 RefreshToken 흐름 정리

```text
로그인 성공
→ RefreshToken 발급
→ Redis에 TTL과 함께 저장
→ AccessToken 만료 후 재발급 요청
→ JWT 만료 여부 확인
→ Redis 존재 여부 확인
→ 둘 다 통과하면 새 토큰 발급
```

---

# 5. Redis TTL 적용 구조 3 — OAuth 가입 중 임시 사용자 정보

## 5.1 목적

OAuth 로그인을 하면 바로 정식 회원가입이 끝나는 경우도 있지만, 추가 정보 입력이 필요한 경우가 있다.

예를 들어 소셜 로그인으로 이메일이나 소셜 식별자는 받았지만, 서비스에서 필요한 전화번호나 생년월일 입력이 아직 안 끝났을 수 있다.

이때 사용자는 완전한 회원도 아니고, 완전히 모르는 사용자도 아니다.

그래서 임시 사용자 정보를 Redis에 저장한다.

하지만 가입을 끝내지 않고 이탈할 수 있으므로 이 정보도 영구 저장하면 안 된다.

그래서 TTL을 둔다.

---

## 5.2 TemporaryUserInfo Redis 엔티티

```java
package com.caring.caringbackend.domain.auth.entity;

import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.redis.core.Indexed;
import org.springframework.data.redis.core.RedisHash;
import org.springframework.data.redis.core.TimeToLive;

import java.time.LocalDate;

@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@RedisHash(value = "temp_user")
public class TemporaryUserInfo {

    @Id
    private String phone;

    @Indexed
    private String accessToken;

    private LocalDate birthDate;

    private String name;

    @TimeToLive
    private Long expiresIn;
}
```

---

## 5.3 코드 설명

```java
@RedisHash(value = "temp_user")
```

임시 사용자 정보를 Redis에 저장한다.

---

```java
@Id
private String phone;
```

전화번호를 key로 사용한다.

같은 전화번호로 재인증하거나 가입을 다시 시도하면 기존 정보가 갱신될 수 있다.

---

```java
@Indexed
private String accessToken;
```

가입 세션을 accessToken 기준으로 찾을 수 있게 한다.

---

```java
@TimeToLive
private Long expiresIn;
```

임시 사용자 정보의 TTL이다.

보통 임시 토큰의 남은 유효시간과 맞춘다.

---

## 5.4 가입 완료 후 즉시 삭제

```java
return transactionTemplate.execute(status -> {
    Member member = Member.builder()
            .name(temporaryUserInfo.getName())
            .phoneNumber(temporaryUserInfo.getPhone())
            .build();

    memberRepository.save(member);
    authCredentialRepository.save(authCredential);

    temporaryUserInfoRepository.delete(temporaryUserInfo);

    return generateTokenByMember(member);
});
```

---

## 5.5 코드 설명

```java
Member member = Member.builder()
        .name(temporaryUserInfo.getName())
        .phoneNumber(temporaryUserInfo.getPhone())
        .build();
```

Redis에 저장해둔 임시 사용자 정보를 기반으로 정식 회원을 만든다.

---

```java
memberRepository.save(member);
authCredentialRepository.save(authCredential);
```

정식 회원 정보와 인증 정보를 RDB에 저장한다.

---

```java
temporaryUserInfoRepository.delete(temporaryUserInfo);
```

회원가입이 완료되었으므로 임시 사용자 정보는 즉시 삭제한다.

TTL 만료를 기다릴 필요가 없다.

---

## 5.6 임시 사용자 정보 흐름 정리

```text
OAuth 로그인
→ 추가 정보 필요
→ TemporaryUserInfo Redis 저장
→ 임시 토큰과 같은 TTL 설정
→ 사용자가 추가 정보 입력 완료
→ 정식 회원 저장
→ 임시 정보 즉시 삭제
```

---

# 6. Redis TTL 전체 요약

| 엔티티               | Redis Key  | TTL 값       | 자동 만료        | 즉시 삭제        |
| ----------------- | ---------- | ----------- | ------------ | ------------ |
| CertificationCode | 전화번호       | yml 기본 300초 | 5분 뒤 소멸      | 인증 성공 시 삭제   |
| RefreshToken      | JWT 토큰 문자열 | JWT 유효시간    | 토큰 만료와 함께 소멸 | 로그아웃 시 삭제 가능 |
| TemporaryUserInfo | 전화번호       | 임시 토큰 남은 시간 | 가입 미완료 시 소멸  | 가입 완료 시 삭제   |

Redis TTL을 통해 일정 시간이 지나면 의미가 없어지는 데이터를 자동으로 정리할 수 있다.

핵심은 다음이다.

```text
짧게 살아야 하는 데이터는 RDB보다 Redis TTL이 더 자연스럽다.
```

---

# 7. 롱폴링

## 7.1 롱폴링이 필요한 이유

채팅 기능에서는 사용자가 새 메시지를 거의 실시간으로 받아야 한다.

가장 단순한 방식은 클라이언트가 일정 시간마다 서버에 새 메시지가 있는지 물어보는 것이다.

예를 들어 1초마다 요청하는 방식이다.

```text
1초마다 GET /messages
1초마다 GET /messages
1초마다 GET /messages
```

이 방식을 짧은 폴링이라고 볼 수 있다.

하지만 메시지가 없어도 계속 요청을 보내기 때문에 비효율적이다.

---

## 7.2 짧은 폴링, 롱폴링, WebSocket 차이

| 방식        | 설명                       | 특징                     |
| --------- | ------------------------ | ---------------------- |
| 짧은 폴링     | 일정 주기로 계속 요청             | 구현 쉬움, 불필요한 요청 많음      |
| 롱폴링       | 서버가 새 데이터가 생길 때까지 잠시 기다림 | 준실시간, WebSocket보다 단순   |
| WebSocket | 연결을 유지하고 양방향 통신          | 실시간성 좋음, 구현과 운영 복잡도 증가 |

롱폴링은 중간 지점에 있다.

WebSocket처럼 완전한 실시간 양방향 연결은 아니지만, 짧은 폴링보다 효율적으로 새 메시지를 받을 수 있다.

---

## 7.3 롱폴링 전체 흐름

```text
1. 클라이언트가 마지막으로 받은 메시지 ID를 함께 서버에 요청한다.

2. 서버는 신규 메시지가 있는지 DB에서 조회한다.

3. 신규 메시지가 있으면 즉시 반환한다.

4. 신규 메시지가 없으면 바로 응답하지 않고 잠시 기다린다.

5. 0.5초 뒤 다시 DB를 조회한다.

6. 최대 30초 동안 반복한다.

7. 30초 안에 새 메시지가 생기면 반환한다.

8. 30초 동안 새 메시지가 없으면 빈 리스트를 반환한다.

9. 클라이언트는 응답을 받으면 다시 poll 요청을 보낸다.
```

---

## 7.4 롱폴링 흐름 그림

```text
클라이언트: GET /poll?lastMessageId=42
                       ↓
서버: 최대 30초 대기 루프 시작
  ├─ 0.0초: DB 조회 → 없음 → 0.5초 대기
  ├─ 0.5초: DB 조회 → 없음 → 0.5초 대기
  ├─ 1.0초: DB 조회 → id=43 발견 → 즉시 반환
  └─ 30.0초: 끝까지 없음 → 빈 리스트 반환

클라이언트:
응답을 받으면 lastMessageId를 갱신하고 다시 poll 요청
```

---

# 8. 롱폴링 Controller 코드

## 8.1 MemberChatController

```java
@GetMapping("/rooms/{chatRoomId}/messages/poll")
@Operation(summary = "4. 롱 폴링", description = "회원이 신규 메시지를 대기합니다. 타임아웃: 30초")
public ResponseEntity<ApiResponse<List<ChatMessageResponse>>> pollMessages(
        @AuthenticationPrincipal MemberDetails memberDetails,
        @PathVariable Long chatRoomId,
        @RequestParam Long lastMessageId) {

    List<ChatMessage> newMessages = chatService.pollMessages(
            chatRoomId,
            lastMessageId,
            memberDetails.getId(),
            SenderType.MEMBER
    );

    List<ChatMessageResponse> messageResponses = newMessages.stream()
            .map(message -> ChatMessageResponse.from(message, getSenderName(message)))
            .collect(Collectors.toList());

    return ResponseEntity.ok(ApiResponse.success("신규 메시지 조회 성공", messageResponses));
}
```

---

## 8.2 코드 설명

```java
@GetMapping("/rooms/{chatRoomId}/messages/poll")
```

채팅방의 신규 메시지를 기다리는 API다.

---

```java
@AuthenticationPrincipal MemberDetails memberDetails
```

현재 로그인한 회원 정보를 가져온다.

이 값으로 누가 요청했는지 확인할 수 있다.

---

```java
@PathVariable Long chatRoomId
```

어느 채팅방의 메시지를 기다릴지 나타낸다.

---

```java
@RequestParam Long lastMessageId
```

클라이언트가 마지막으로 받은 메시지 ID다.

예를 들어 lastMessageId가 42라면 서버는 42보다 큰 메시지만 찾는다.

---

```java
chatService.pollMessages(...)
```

실제 롱폴링 로직은 Service에서 처리한다.

---

```java
ChatMessageResponse.from(...)
```

엔티티를 응답 DTO로 변환한다.

---

# 9. 롱폴링 Service 코드

## 9.1 ChatService

```java
public List<ChatMessage> pollMessages(
        Long chatRoomId,
        Long lastMessageId,
        Long userId,
        SenderType userType
) {
    getChatRoomWithPermission(chatRoomId, userId, userType);

    long startTime = System.currentTimeMillis();
    long timeout = 30000;
    long pollingInterval = 500;

    while (System.currentTimeMillis() - startTime < timeout) {

        List<ChatMessage> newMessages =
                chatMessageRepository.findNewMessages(chatRoomId, lastMessageId);

        if (!newMessages.isEmpty()) {
            log.debug("신규 메시지 발견: chatRoomId={}, count={}", chatRoomId, newMessages.size());
            return newMessages;
        }

        try {
            Thread.sleep(pollingInterval);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.warn("롱 폴링 중단: chatRoomId={}", chatRoomId);
            break;
        }
    }

    log.debug("롱 폴링 타임아웃: chatRoomId={}, lastMessageId={}", chatRoomId, lastMessageId);
    return List.of();
}
```

---

## 9.2 코드 설명

```java
getChatRoomWithPermission(chatRoomId, userId, userType);
```

롱폴링을 시작하기 전에 채팅방 접근 권한을 확인한다.

채팅은 민감한 대화 데이터이기 때문에 본인 채팅방이 아니면 조회하면 안 된다.

---

```java
long startTime = System.currentTimeMillis();
long timeout = 30000;
long pollingInterval = 500;
```

롱폴링 시간 설정이다.

| 변수              | 의미             |
| --------------- | -------------- |
| startTime       | 요청 시작 시간       |
| timeout         | 최대 대기 시간 30초   |
| pollingInterval | DB 재조회 간격 0.5초 |

---

```java
while (System.currentTimeMillis() - startTime < timeout)
```

요청 시작 후 30초가 지나기 전까지 반복한다.

---

```java
chatMessageRepository.findNewMessages(chatRoomId, lastMessageId)
```

DB에서 lastMessageId보다 큰 메시지를 찾는다.

즉 클라이언트가 아직 받지 못한 신규 메시지만 조회한다.

---

```java
if (!newMessages.isEmpty()) {
    return newMessages;
}
```

신규 메시지가 있으면 30초를 다 기다리지 않고 즉시 반환한다.

이게 롱폴링의 핵심이다.

---

```java
Thread.sleep(pollingInterval);
```

신규 메시지가 없으면 0.5초 기다렸다가 다시 조회한다.

---

```java
return List.of();
```

30초 동안 신규 메시지가 없으면 빈 리스트를 반환한다.

클라이언트는 빈 리스트를 받아도 다시 poll 요청을 보내면 된다.

---

# 10. 신규 메시지 조회 Repository 코드

```java
@Query("""
        SELECT cm FROM ChatMessage cm
        WHERE cm.chatRoom.id = :chatRoomId
        AND cm.id > :lastMessageId
        AND cm.deleted = false
        ORDER BY cm.createdAt ASC
        """)
List<ChatMessage> findNewMessages(
        @Param("chatRoomId") Long chatRoomId,
        @Param("lastMessageId") Long lastMessageId
);
```

---

## 10.1 코드 설명

```java
WHERE cm.chatRoom.id = :chatRoomId
```

특정 채팅방의 메시지만 조회한다.

---

```java
AND cm.id > :lastMessageId
```

클라이언트가 마지막으로 받은 메시지 이후의 메시지만 조회한다.

이 조건이 없으면 매번 전체 메시지를 다시 가져오게 된다.

---

```java
AND cm.deleted = false
```

Soft Delete된 메시지는 제외한다.

---

```java
ORDER BY cm.createdAt ASC
```

오래된 메시지부터 정렬한다.

채팅은 시간 순서대로 보여줘야 하기 때문이다.

---

# 11. 롱폴링 권한 검증 코드

```java
private ChatRoom getChatRoomWithPermission(Long chatRoomId, Long userId, SenderType userType) {
    if (userType == SenderType.MEMBER) {
        return chatRoomRepository.findByIdAndMemberId(chatRoomId, userId)
                .orElseThrow(() -> new BusinessException(ErrorCode.CHAT_ACCESS_DENIED));

    } else if (userType == SenderType.INSTITUTION_ADMIN) {
        InstitutionAdmin admin = institutionAdminRepository.findById(userId)
                .orElseThrow(() -> new BusinessException(ErrorCode.ADMIN_NOT_FOUND));

        return chatRoomRepository.findByIdAndInstitutionId(
                        chatRoomId,
                        admin.getInstitution().getId()
                )
                .orElseThrow(() -> new BusinessException(ErrorCode.CHAT_ACCESS_DENIED));
    }

    throw new BusinessException(ErrorCode.INVALID_SENDER_TYPE);
}
```

---

## 11.1 코드 설명

회원이 요청한 경우:

```java
chatRoomRepository.findByIdAndMemberId(chatRoomId, userId)
```

해당 채팅방이 이 회원의 채팅방인지 확인한다.

---

기관 관리자가 요청한 경우:

```java
institutionAdminRepository.findById(userId)
```

먼저 기관 관리자를 찾는다.

그 다음:

```java
chatRoomRepository.findByIdAndInstitutionId(chatRoomId, admin.getInstitution().getId())
```

이 관리자가 속한 기관의 채팅방인지 확인한다.

---

## 11.2 왜 권한 검증이 중요한가

롱폴링은 30초 동안 연결을 유지한다.

만약 권한 검증 없이 시작하면, 다른 사람의 채팅방 메시지를 기다릴 수도 있다.

그래서 롱폴링 시작 전에 반드시 본인이 접근 가능한 채팅방인지 확인해야 한다.

---

# 12. 롱폴링의 장점과 한계

## 12.1 장점

* WebSocket보다 구현이 쉽다.
* HTTP 요청/응답 구조만으로 구현 가능하다.
* 메시지 빈도가 낮은 서비스에서는 충분히 사용할 수 있다.
* 서버가 새 메시지가 생길 때까지 기다렸다가 반환하므로 짧은 폴링보다 요청 낭비가 적다.

---

## 12.2 한계

* 요청 하나가 최대 30초 동안 서버 스레드를 점유할 수 있다.
* 코드에서는 0.5초마다 DB를 조회하므로 사용자가 많아지면 DB 부하가 생길 수 있다.
* 완전한 실시간 양방향 통신은 아니다.
* 채팅 사용량이 많아지면 WebSocket이나 Redis Pub/Sub 구조로 개선하는 것이 좋다.

---

## 12.3 개선 방향

현재 구조:

```text
롱폴링 요청
→ 0.5초마다 DB 조회
→ 신규 메시지 있으면 반환
```

개선 가능한 구조:

```text
메시지 저장
→ Redis Pub/Sub 이벤트 발행
→ 대기 중인 요청에 즉시 응답
```

이렇게 바꾸면 DB를 반복 조회하지 않고 이벤트 기반으로 처리할 수 있다.

---

## 12.4 면접 답변 포인트

```text
요양원 상담처럼 메시지 빈도가 아주 높지 않은 서비스에서는 WebSocket 연결을 계속 유지하는 것보다 롱폴링이 단순하고 적절할 수 있다고 판단했습니다.

롱폴링은 서버가 신규 메시지가 생길 때까지 최대 30초간 요청을 유지하다가, 메시지가 생기면 즉시 반환하는 방식입니다.

다만 0.5초 간격으로 DB를 조회하는 구조라 사용자가 많아지면 DB 부하와 스레드 점유 문제가 생길 수 있습니다.

이 한계를 알고 있으며, 개선한다면 Redis Pub/Sub이나 WebSocket으로 이벤트 기반 구조를 적용할 수 있습니다.
```

---

# 13. FastAPI 연동

## 13.1 FastAPI 연동이 필요한 이유

Spring 백엔드에서 모든 AI 예측 로직을 직접 처리하지 않고, Python 기반 FastAPI 서버를 따로 두는 경우가 많다.

이유는 다음과 같다.

* AI 모델은 Python 생태계에서 많이 사용된다.
* FastAPI는 Python으로 API 서버를 빠르게 만들 수 있다.
* Spring은 서비스 흐름, 인증, DB 저장, 알림을 담당한다.
* FastAPI는 예측 모델 실행을 담당한다.

즉 역할을 나누는 구조다.

```text
Spring
= 서비스 흐름 담당

FastAPI
= AI 모델 예측 담당
```

---

## 13.2 전체 흐름

Monitory 프로젝트의 FastAPI 연동 흐름은 다음과 같다.

```text
S3에 설비 데이터 JSON 업로드
→ S3 ObjectCreated 이벤트 발생
→ SQS로 이벤트 전달
→ Spring S3EventSqsListener가 SQS 메시지 수신
→ S3 key에서 zoneId, equipId 추출
→ EquipPredictProcessor 호출
→ FastAPI /api/v1/predict 호출
→ 예측 결과 수신
→ 예상 점검일 계산
→ EquipHistory 저장
→ Slack 알림 조건 확인
→ AbnormalLog 저장
```

이 구조에서 중요한 점은 Spring이 직접 AI 예측을 수행하지 않는다는 점이다.

Spring은 FastAPI를 호출하고, 그 결과를 서비스 흐름에 연결한다.

---

# 14. RestTemplate Bean 등록

## 14.1 코드

```java
package com.factoreal.backend.global.config;

import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.build();
    }
}
```

---

## 14.2 코드 설명

```java
@Configuration
```

설정 클래스다.

---

```java
@Bean
public RestTemplate restTemplate(...)
```

RestTemplate 객체를 Spring Bean으로 등록한다.

---

```java
return builder.build();
```

RestTemplateBuilder를 사용해 RestTemplate을 생성한다.

---

## 14.3 왜 Bean으로 등록하는가

RestTemplate은 Spring에서 HTTP 요청을 보내기 위한 객체다.

FastAPI는 HTTP 서버이므로 Spring에서 FastAPI를 호출하려면 HTTP Client가 필요하다.

RestTemplate을 Bean으로 등록하면 필요한 서비스에서 다음처럼 주입받아 사용할 수 있다.

```java
private final RestTemplate restTemplate;
```

직접 new로 만들 수도 있지만, Bean으로 등록하면 공통 설정과 테스트 관리가 쉬워진다.

---

# 15. SQS 이벤트 수신

## 15.1 S3EventSqsListener 코드

```java
@SqsListener("${aws.sqs.queue-url}")
public void handleMessage(String rawMessage) {

    JsonNode root = objectMapper.readTree(rawMessage);

    for (JsonNode rec : events) {
        String eventName = rec.path("eventName").asText("");

        if (!eventName.startsWith("ObjectCreated")) {
            continue;
        }

        JsonNode s3 = rec.path("s3");
        String bucket = s3.path("bucket").path("name").asText("");
        String keyEnc = s3.path("object").path("key").asText("");
        String key = URLDecoder.decode(keyEnc, StandardCharsets.UTF_8);

        if (key.startsWith("EQUIPMENT/") && key.endsWith(".json")) {
            String zoneId = null;
            String equipId = null;

            for (String segment : key.split("/")) {
                if (segment.startsWith("zone_id=")) {
                    zoneId = segment.substring("zone_id=".length());
                } else if (segment.startsWith("equip_id=")) {
                    equipId = segment.substring("equip_id=".length());
                }
            }

            if (zoneId != null && equipId != null) {
                equipPredictProcessor.equipPredProcess(zoneId, equipId);
            }
        }
    }
}
```

---

## 15.2 @SqsListener

```java
@SqsListener("${aws.sqs.queue-url}")
```

SQS 큐에 메시지가 들어오면 이 메서드가 실행된다.

즉 직접 주기적으로 SQS를 확인하지 않아도 된다.

Spring Cloud AWS가 큐를 구독하고 있다가 메시지가 오면 handleMessage를 호출한다.

---

## 15.3 rawMessage

```java
public void handleMessage(String rawMessage)
```

SQS에서 받은 원본 메시지다.

이 메시지 안에는 S3 이벤트 정보가 JSON 형태로 들어 있다.

---

## 15.4 ObjectCreated 이벤트만 처리

```java
if (!eventName.startsWith("ObjectCreated")) {
    continue;
}
```

S3 이벤트 중 파일 생성 이벤트만 처리한다.

파일 삭제나 다른 이벤트는 무시한다.

---

## 15.5 key에서 zoneId, equipId 추출

```java
for (String segment : key.split("/")) {
    if (segment.startsWith("zone_id=")) {
        zoneId = segment.substring("zone_id=".length());
    } else if (segment.startsWith("equip_id=")) {
        equipId = segment.substring("equip_id=".length());
    }
}
```

S3 key가 다음 형태라고 가정한다.

```text
EQUIPMENT/zone_id=Z01/equip_id=E01/data.json
```

이 문자열을 / 기준으로 나누면 다음처럼 된다.

```text
EQUIPMENT
zone_id=Z01
equip_id=E01
data.json
```

이 중 zone_id=로 시작하는 부분에서 Z01을 꺼내고, equip_id=로 시작하는 부분에서 E01을 꺼낸다.

---

# 16. FastAPI URL 설정

## 16.1 application.yml

```yaml
fastapi:
  base-url: ${FASTAPI_URL:http://localhost:8000}
  predict-endpoint: /api/v1/predict
```

---

## 16.2 application-local.yml

```yaml
fastapi:
  base-url: http://localhost:8000
```

---

## 16.3 application-cloud.yml

```yaml
fastapi:
  base-url: ${FASTAPI_URL}
```

---

## 16.4 왜 yml로 분리하는가

로컬 개발 환경과 클라우드 배포 환경에서는 FastAPI 서버 주소가 다를 수 있다.

로컬에서는 다음 주소를 쓸 수 있다.

```text
http://localhost:8000
```

클라우드에서는 환경변수로 주입한다.

```text
FASTAPI_URL
```

이렇게 하면 코드 수정 없이 환경별 주소만 바꿀 수 있다.

---

# 17. EquipPredictProcessor URL 조립

## 17.1 코드

```java
@Value("${fastapi.base-url}")
private String fastApiBaseUrl;

@Value("${fastapi.predict-endpoint}")
private String predictEndpoint;

String url = UriComponentsBuilder
        .fromUriString(fastApiBaseUrl)
        .path(predictEndpoint)
        .queryParam("equipId", equipId)
        .queryParam("zoneId", zoneId)
        .toUriString();
```

---

## 17.2 코드 설명

```java
@Value("${fastapi.base-url}")
private String fastApiBaseUrl;
```

yml에 있는 FastAPI 서버 기본 주소를 가져온다.

예시:

```text
http://localhost:8000
```

---

```java
@Value("${fastapi.predict-endpoint}")
private String predictEndpoint;
```

예측 API 경로를 가져온다.

예시:

```text
/api/v1/predict
```

---

```java
UriComponentsBuilder
        .fromUriString(fastApiBaseUrl)
        .path(predictEndpoint)
        .queryParam("equipId", equipId)
        .queryParam("zoneId", zoneId)
        .toUriString();
```

FastAPI 호출 URL을 만든다.

결과는 다음과 같은 형태다.

```text
http://localhost:8000/api/v1/predict?equipId=E01&zoneId=Z01
```

---

## 17.3 왜 문자열 더하기를 하지 않는가

아래처럼 직접 문자열을 붙일 수도 있다.

```java
String url = fastApiBaseUrl + predictEndpoint + "?equipId=" + equipId + "&zoneId=" + zoneId;
```

하지만 실무에서는 UriComponentsBuilder를 쓰는 것이 안전하다.

이유는 다음과 같다.

* ?와 &를 실수할 수 있다.
* query parameter 인코딩 문제가 생길 수 있다.
* URL 조립 로직이 길어질수록 가독성이 떨어진다.
* 파라미터가 늘어나도 관리하기 쉽다.

---

# 18. RestTemplate으로 FastAPI 호출

## 18.1 코드

```java
log.info("FastAPI 호출 - URL: {}", url);

ResponseEntity<MaintenancePredictionResponse> resp =
        restTemplate.getForEntity(url, MaintenancePredictionResponse.class);

if (resp.getBody() == null || resp.getBody().getPredictions().isEmpty()) {
    log.warn("FastAPI 예측값 없음 (equipId={})", equipId);
    return;
}
```

---

## 18.2 getForEntity

```java
restTemplate.getForEntity(url, MaintenancePredictionResponse.class)
```

GET 요청을 보내고 응답을 ResponseEntity로 받는다.

두 번째 인자인 MaintenancePredictionResponse.class는 응답 JSON을 어떤 Java 객체로 바꿀지 알려주는 역할이다.

---

## 18.3 ResponseEntity

ResponseEntity는 HTTP 응답 전체를 담는 객체다.

여기에는 다음 정보가 들어갈 수 있다.

* HTTP 상태 코드
* 응답 헤더
* 응답 바디

이 코드에서는 응답 바디에 있는 예측 결과를 사용한다.

---

## 18.4 null 방어 코드

```java
if (resp.getBody() == null || resp.getBody().getPredictions().isEmpty()) {
    return;
}
```

FastAPI 서버가 정상 응답을 주지 않거나 predictions가 비어 있으면 이후 로직을 진행하지 않는다.

이런 방어 코드가 없으면 NullPointerException이 발생할 수 있다.

---

# 19. FastAPI 응답 DTO

## 19.1 FastAPI 응답 예시

```json
{
  "status": "ok",
  "predictions": [4.0]
}
```

---

## 19.2 Java DTO

```java
package com.factoreal.backend.domain.equip.dto;

import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.util.List;

@Getter
@Setter
@NoArgsConstructor
public class MaintenancePredictionResponse {

    private String status;

    private List<Double> predictions;
}
```

---

## 19.3 DTO 설명

```java
private String status;
```

FastAPI 응답 상태다.

예를 들어 ok, error 같은 값이 들어갈 수 있다.

---

```java
private List<Double> predictions;
```

예측 결과 배열이다.

예를 들어 [4.0]이면 잔존수명이 4일이라는 뜻으로 해석한다.

---

## 19.4 자동 역직렬화

```java
restTemplate.getForEntity(url, MaintenancePredictionResponse.class)
```

이 코드를 실행하면 Jackson이 JSON 응답을 MaintenancePredictionResponse 객체로 자동 변환한다.

이 과정을 역직렬화라고 한다.

```text
JSON 문자열
→ Java 객체
```

---

# 20. FastAPI 예측 결과 처리

## 20.1 코드

```java
int remainDays = resp.getBody().getPredictions().get(0).intValue();

LocalDate expectedMaintenanceDate =
        equipMaintenanceService.calculateExpectedMaintenanceDate(remainDays);

equipMaintenanceService.processMaintenancePrediction(equipId, expectedMaintenanceDate);

if (slackEquipAlarmService.shouldSendAlert(expectedMaintenanceDate)) {
    slackEquipAlarmService.sendEquipmentMaintenanceAlert(
            equip.getEquipName(),
            zone.getZoneName(),
            expectedMaintenanceDate,
            daysUntil
    );
}

int dangerLevel = remainDays <= 3 ? 2 : (remainDays < 5 ? 1 : 0);

if (dangerLevel > 0) {
    abnormalLogService.saveEquipAbnormalLog(zone, equip, remainDays, dangerLevel);
}
```

---

## 20.2 remainDays

```java
int remainDays = resp.getBody().getPredictions().get(0).intValue();
```

FastAPI 응답의 첫 번째 예측값을 꺼낸다.

예를 들어 predictions가 [4.0]이면 remainDays는 4가 된다.

---

## 20.3 예상 점검일 계산

```java
LocalDate expectedMaintenanceDate =
        equipMaintenanceService.calculateExpectedMaintenanceDate(remainDays);
```

잔존수명이 4일이면 오늘 날짜에 4일을 더해 예상 점검일을 계산한다.

```text
오늘 날짜 + remainDays
= 예상 점검일
```

---

## 20.4 점검 예측 이력 저장

```java
equipMaintenanceService.processMaintenancePrediction(equipId, expectedMaintenanceDate);
```

예상 점검일을 DB에 저장하거나 기존 이력과 비교하는 로직으로 연결한다.

---

## 20.5 Slack 알림

```java
if (slackEquipAlarmService.shouldSendAlert(expectedMaintenanceDate)) {
    slackEquipAlarmService.sendEquipmentMaintenanceAlert(...);
}
```

예상 점검일이 D-5, D-3 같은 조건에 해당하면 Slack 알림을 보낸다.

즉 FastAPI 예측 결과가 단순히 화면에만 쓰이는 것이 아니라 운영 알림으로 이어진다.

---

## 20.6 위험 등급 계산

```java
int dangerLevel = remainDays <= 3 ? 2 : (remainDays < 5 ? 1 : 0);
```

잔존수명에 따라 위험 등급을 계산한다.

예시:

| remainDays | dangerLevel | 의미 |
| ---------- | ----------- | -- |
| 3일 이하      | 2           | 심각 |
| 4일         | 1           | 경고 |
| 5일 이상      | 0           | 정상 |

---

## 20.7 이상 로그 저장

```java
if (dangerLevel > 0) {
    abnormalLogService.saveEquipAbnormalLog(zone, equip, remainDays, dangerLevel);
}
```

위험 등급이 0보다 크면 AbnormalLog를 저장한다.

즉 설비 점검 예측 결과가 위험하다고 판단되면 이상 로그로 기록된다.

---

# 21. FastAPI 연동 전체 흐름 정리

```text
S3에 EQUIPMENT/zone_id=Z01/equip_id=E01/data.json 업로드
↓
S3 ObjectCreated 이벤트 발생
↓
SQS 메시지 전달
↓
S3EventSqsListener.handleMessage 실행
↓
S3 key에서 zoneId, equipId 추출
↓
EquipPredictProcessor.equipPredProcess 호출
↓
UriComponentsBuilder로 FastAPI URL 생성
↓
RestTemplate.getForEntity로 FastAPI 호출
↓
FastAPI 응답 JSON 수신
↓
MaintenancePredictionResponse DTO로 자동 변환
↓
remainDays 추출
↓
예상 점검일 계산
↓
EquipHistory 저장
↓
Slack 알림 조건 확인
↓
위험 등급 계산
↓
AbnormalLog 저장
```

---

# 22. FastAPI 연동 면접 답변 포인트

```text
설비 데이터가 S3에 업로드되면 S3 ObjectCreated 이벤트가 SQS로 전달되고, Spring의 S3EventSqsListener가 이를 수신하도록 구성했습니다.

리스너에서는 S3 key에 포함된 zone_id와 equip_id를 파싱한 뒤 EquipPredictProcessor로 전달했습니다.

Processor에서는 yml에 분리된 FastAPI 서버 주소와 예측 endpoint를 @Value로 주입받고, UriComponentsBuilder로 안전하게 URL을 조립했습니다.

이후 RestTemplate으로 FastAPI의 /api/v1/predict API를 호출하고, 응답 JSON을 MaintenancePredictionResponse DTO로 역직렬화했습니다.

응답으로 받은 잔존수명 예측값은 예상 점검일 계산, DB 이력 저장, 조건부 Slack 알림, 이상 로그 저장으로 이어지도록 서비스 흐름에 연결했습니다.
```

---

# 23. 이번 주에 이해한 핵심 정리

## 23.1 Redis TTL

Redis TTL은 일정 시간이 지나면 자동으로 사라져야 하는 데이터를 관리할 때 사용한다.

휴대폰 인증번호, RefreshToken, 임시 사용자 정보처럼 만료 개념이 중요한 데이터에 적합하다.

---

## 23.2 @TimeToLive

@TimeToLive가 붙은 필드 값이 Redis TTL로 사용된다.

```java
@TimeToLive
private Long expiresIn;
```

expiresIn이 300이면 300초 뒤 Redis에서 자동 삭제된다.

---

## 23.3 인증번호 TTL

인증번호는 5분 TTL로 저장하고, 인증 성공 시 즉시 삭제한다.

---

## 23.4 RefreshToken TTL

RefreshToken은 JWT 유효시간과 Redis TTL을 맞춘다.

재발급 시 JWT 만료 여부와 Redis 존재 여부를 모두 확인한다.

---

## 23.5 임시 사용자 정보 TTL

OAuth 가입 중 필요한 임시 정보는 Redis에 저장하고, 가입 완료 시 즉시 삭제한다.

가입을 완료하지 않으면 TTL 만료로 자동 삭제된다.

---

## 23.6 롱폴링

롱폴링은 클라이언트 요청을 서버가 바로 응답하지 않고, 새 데이터가 생길 때까지 일정 시간 기다리는 방식이다.

채팅처럼 메시지 빈도가 높지 않은 서비스에서 WebSocket보다 단순한 대안으로 사용할 수 있다.

---

## 23.7 lastMessageId

lastMessageId는 클라이언트가 마지막으로 받은 메시지 ID다.

서버는 이 값보다 큰 메시지만 조회해서 신규 메시지만 반환한다.

---

## 23.8 롱폴링 한계

현재 구조는 0.5초마다 DB를 조회하므로 사용자가 많아지면 DB 부하가 생길 수 있다.

개선한다면 Redis Pub/Sub이나 WebSocket으로 이벤트 기반 구조를 적용할 수 있다.

---

## 23.9 FastAPI 연동

Spring은 서비스 흐름을 담당하고, FastAPI는 AI 예측을 담당한다.

Spring은 RestTemplate으로 FastAPI를 호출하고, 결과를 DTO로 받아 후속 비즈니스 로직에 연결한다.

---

## 23.10 SQS 기반 트리거

FastAPI 호출은 사용자가 직접 버튼을 눌러 실행되는 것이 아니라, S3에 설비 JSON이 업로드된 이벤트를 SQS로 받아 시작된다.

즉 이벤트 기반 비동기 파이프라인이다.

---

# 24. 이번 주에 배운 점

1. Redis TTL은 만료 시간이 중요한 데이터를 자동으로 정리하기 위해 사용한다.

2. @RedisHash는 Redis 저장 엔티티를 의미하고, @TimeToLive는 자동 만료 시간을 의미한다.

3. 휴대폰 인증번호는 TTL로 자동 만료시키고, 인증 성공 시 즉시 삭제하는 것이 안전하다.

4. RefreshToken은 JWT 만료시간과 Redis TTL을 맞춰 관리할 수 있다.

5. Redis에 RefreshToken이 없으면 JWT가 남아 있어도 재발급을 막을 수 있다.

6. OAuth 가입 중 임시 사용자 정보는 Redis TTL로 관리하면 가입 중단 시 자동 정리된다.

7. 롱폴링은 WebSocket보다 단순하지만 준실시간 응답을 만들 수 있는 방식이다.

8. lastMessageId를 기준으로 신규 메시지만 조회하면 중복 응답을 줄일 수 있다.

9. 롱폴링은 서버 스레드 점유와 DB 반복 조회라는 한계가 있다.

10. FastAPI 연동에서는 Spring과 Python AI 서버의 역할을 분리한다.

11. RestTemplate을 Bean으로 등록하면 여러 서비스에서 주입받아 사용할 수 있다.

12. UriComponentsBuilder는 URL을 안전하게 조립할 때 사용한다.

13. ResponseEntity는 HTTP 응답 전체를 담고, DTO는 응답 body를 Java 객체로 받기 위해 사용한다.

14. FastAPI 예측 결과는 단순 조회가 아니라 DB 저장, Slack 알림, 이상 로그 저장으로 연결될 수 있다.

15. S3 이벤트, SQS, Spring Listener, FastAPI 호출을 연결하면 데이터 업로드 이후 예측 처리를 자동화할 수 있다.

---

# 25. 다음에 더 보면 좋은 것

* Redis Repository가 실제로 어떤 메서드를 제공하는지
* Redis TTL 만료 후 조회 결과가 어떻게 달라지는지
* 로그아웃 시 RefreshToken을 삭제하는 흐름
* WebSocket과 롱폴링의 운영 비용 차이
* Redis Pub/Sub으로 롱폴링 DB 조회를 줄이는 방법
* SQS 메시지 삭제와 재처리 구조
* RestTemplate과 WebClient 차이
* FastAPI 장애 시 재시도와 예외 처리 전략
* 예측 실패 시 Slack 알림이나 보상 트랜잭션을 어떻게 설계할지