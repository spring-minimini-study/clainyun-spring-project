# Week7 — S3 Presigned URL & AWS 인증 구조 정리

## 7주차

이번 주에는 S3 Presigned URL을 중심으로 AWS S3 파일 업로드 구조와 Spring에서 AWS SDK를 사용하는 방식을 정리했다.

처음에는 S3 Presigned URL을 단순히 S3에 파일을 올리는 URL이라고 생각했다. 하지만 실제로는 단순 URL이 아니라 AWS가 정해진 조건과 만료시간을 기준으로 서명한 임시 권한 URL이라는 점을 이해했다.

이번 주에 정리한 핵심 내용은 다음과 같다.

- 기존 파일 업로드 방식의 문제점
- S3 Presigned URL을 사용하게 된 이유
- Presigned URL 사용 시 전체 흐름
- S3, Bucket, Key 개념
- S3Presigner, PutObjectRequest, PutObjectPresignRequest 역할
- UUID를 파일명에 붙이는 이유
- URL을 개발자가 직접 만드는 것인지, AWS SDK가 만드는 것인지
- cloud.aws.region.static 설정 의미
- @Bean으로 외부 라이브러리 객체를 등록하는 이유
- AWS access-key, secret-key를 어디서 발급받는지
- 실무에서 환경변수와 IAM Role을 사용하는 이유
- 현재 Monitory 프로젝트의 S3 이벤트 기반 구조와 Presigned URL의 차이
- Monitory 프로젝트에 Presigned URL을 적용한다면 어떤 코드 구조가 되는지

---

## 1. 기존 파일 업로드 방식

Presigned URL을 이해하려면 먼저 기존 파일 업로드 방식부터 봐야 한다.

일반적으로 파일 업로드는 다음 흐름으로 진행된다.

```text
브라우저
→ Spring 서버
→ S3
````

단계별로 보면 다음과 같다.

```text
1. 사용자가 브라우저에서 파일을 선택한다.

2. 브라우저가 Spring 서버로 파일을 전송한다.

3. Spring 서버가 파일을 받은 뒤 S3에 다시 업로드한다.

4. Spring 서버는 S3에 저장된 파일 경로를 DB에 저장한다.
```

그림으로 보면 다음과 같다.

```text
사용자
  ↓ 파일 선택
브라우저
  ↓ multipart/form-data
Spring 서버
  ↓ AWS SDK로 업로드
S3
  ↓
DB에는 파일 key 저장
```

---

## 2. 기존 업로드 방식의 문제점

기존 방식은 구조가 단순하지만 서버가 부담을 많이 받는다.

예를 들어 사용자가 500MB 영상을 업로드한다고 하면 파일이 서버를 한 번 거쳐간다.

```text
브라우저 → Spring 서버
Spring 서버 → S3
```

즉 같은 파일이 네트워크를 두 번 탄다.

이때 서버에 생기는 문제는 다음과 같다.

* 대용량 파일을 직접 받아야 한다.
* 업로드 요청이 많아지면 서버 네트워크 사용량이 커진다.
* 서버 메모리나 임시 저장 공간 사용량이 늘어날 수 있다.
* 파일 업로드 중 서버 장애가 나면 전체 업로드가 실패한다.
* 파일 업로드 처리 때문에 API 서버의 다른 요청 처리도 느려질 수 있다.
* 서버가 파일 저장 역할까지 맡으면서 책임이 커진다.

그래서 실무에서는 대용량 파일 업로드를 할 때 서버가 파일을 직접 받지 않도록 설계하는 경우가 많다.

---

## 3. S3 Presigned URL을 사용하게 된 이유

Presigned URL은 위 문제를 줄이기 위해 사용한다.

핵심은 Spring 서버가 파일을 직접 받지 않게 하는 것이다.

기존 방식은 다음과 같았다.

```text
브라우저 → Spring 서버 → S3
```

Presigned URL 방식은 다음처럼 바뀐다.

```text
브라우저 → Spring 서버
브라우저 → S3
```

Spring 서버는 파일을 받지 않는다.
대신 S3에 직접 업로드할 수 있는 임시 URL만 만들어서 프론트에게 준다.

이후 프론트는 그 URL로 S3에 직접 파일을 업로드한다.

---

## 4. S3 Presigned URL 한 줄 정의

S3 Presigned URL은 S3에 직접 업로드하거나 다운로드할 수 있도록 AWS 서명이 붙은 임시 URL이다.

핵심은 세 가지다.

```text
1. S3에 직접 접근한다.

2. AWS Signature가 붙어 있다.

3. 정해진 시간 동안만 사용할 수 있다.
```

즉 Presigned URL은 누구나 영구적으로 접근할 수 있는 공개 URL이 아니다.

정해진 시간 동안만 사용할 수 있고, 정해진 파일 위치에 대해서만 사용할 수 있는 임시 권한 URL이다.

---

## 5. S3 Presigned URL을 사용하면 좋아지는 점

| 장점            | 설명                              |
| ------------- | ------------------------------- |
| 서버 부하 감소      | 파일이 Spring 서버를 거치지 않는다.         |
| 네트워크 비용 감소    | 서버가 파일을 다시 S3로 보내지 않아도 된다.      |
| 대용량 파일 처리에 유리 | 이미지, 영상, 첨부파일 업로드에 적합하다.        |
| 업로드 속도 개선     | 클라이언트가 S3로 직접 전송한다.             |
| 서버 책임 분리      | 서버는 권한 발급과 DB 저장에 집중한다.         |
| 보안 제어 가능      | URL 만료시간과 업로드 대상 key를 제한할 수 있다. |

정리하면 Presigned URL은 파일 전송 부담을 S3로 넘기고, Spring 서버는 업로드 권한 발급과 데이터 관리만 담당하게 만드는 구조다.

---

## 6. S3 기본 개념

### 6.1 S3

S3는 AWS에서 제공하는 파일 저장 서비스다.

이미지, 영상, JSON, CSV, PDF 같은 파일을 저장할 수 있다.

일반 서버에도 파일을 저장할 수 있지만, 실무에서는 파일 저장을 서버 내부가 아니라 S3 같은 외부 저장소에 맡기는 경우가 많다.

서버에 직접 파일을 저장하면 다음 문제가 생긴다.

* 서버를 재배포할 때 파일 관리가 복잡해진다.
* 서버가 여러 대로 늘어나면 어느 서버에 파일이 있는지 관리해야 한다.
* 대용량 파일이 많아지면 서버 디스크가 부담된다.
* 파일 백업, 접근 제어, 보안 설정을 직접 관리해야 한다.

그래서 파일은 S3에 저장하고, Spring 서버는 파일 key만 DB에 저장하는 구조를 많이 사용한다.

---

### 6.2 Bucket

Bucket은 S3에서 파일을 담는 최상위 저장 공간이다.

예시:

```text
monitory-equipment-bucket
caring-image-bucket
```

비유하면 컴퓨터의 큰 드라이브나 최상위 폴더 같은 느낌이다.

```text
S3
└── bucket
    └── file
```

---

### 6.3 Key

Key는 S3 안에서 파일의 위치를 나타내는 문자열이다.

예시:

```text
profile/550e8400-profile.png
EQUIPMENT/zone_id=Z001/equip_id=E001/data.json
```

중요한 점은 S3에는 실제 폴더가 있는 것이 아니라는 점이다.

S3는 아래 전체를 하나의 key 문자열로 본다.

```text
EQUIPMENT/zone_id=Z001/equip_id=E001/data.json
```

다만 중간에 /가 들어가 있어서 콘솔 화면에서 폴더처럼 보일 뿐이다.

정리하면 다음과 같다.

```text
bucket = 파일 저장소 이름
key = 그 저장소 안에서의 파일 경로
```

---

## 7. Presigned URL 전체 흐름

프로필 이미지를 업로드하는 상황으로 보면 흐름은 다음과 같다.

```text
1. 프론트가 Spring 서버에 요청한다.
   profile.png 올릴 수 있는 URL 주세요.

2. Spring 서버가 S3Presigner를 사용해 Presigned URL을 생성한다.

3. Spring 서버가 프론트에게 uploadUrl과 key를 응답한다.

4. 프론트는 uploadUrl로 S3에 직접 PUT 요청을 보낸다.

5. S3는 URL에 포함된 Signature를 검증한다.

6. Signature가 맞고 만료 시간이 지나지 않았으면 업로드를 허용한다.

7. 업로드가 끝나면 프론트는 key를 서버에 다시 전달한다.

8. Spring 서버는 DB에 key를 저장한다.
```

그림으로 보면 다음과 같다.

```text
[1] 프론트 → Spring
    파일명과 파일 타입을 보내며 업로드 URL 요청

[2] Spring
    bucket, key, 만료시간을 기준으로 Presigned URL 생성

[3] Spring → 프론트
    uploadUrl, key 반환

[4] 프론트 → S3
    uploadUrl로 파일 직접 업로드

[5] S3
    Signature 검증 후 저장

[6] 프론트 → Spring
    업로드 완료 후 key 전달

[7] Spring → DB
    key 저장
```

핵심은 다음이다.

```text
Spring 서버는 파일을 직접 받지 않는다.
Spring 서버는 업로드 가능한 임시 URL만 발급한다.
```

---

## 8. 실제 Spring 패키지 구조 예시

Presigned URL 기능을 구현한다면 보통 다음 클래스들이 생긴다.

```text
global/config/S3Config.java
domain/file/controller/FileController.java
domain/file/service/S3PresignedUrlService.java
domain/file/dto/PresignedUrlResponse.java
```

각 클래스 역할은 다음과 같다.

| 클래스                   | 역할                            |
| --------------------- | ----------------------------- |
| S3Config              | S3Presigner를 Spring Bean으로 등록 |
| FileController        | 프론트 요청을 받는 API                |
| S3PresignedUrlService | Presigned URL 생성 로직 담당        |
| PresignedUrlResponse  | uploadUrl과 key 응답 DTO         |

---

## 9. build.gradle 의존성

S3 Presigned URL을 사용하려면 AWS SDK S3 의존성이 필요하다.

```gradle
implementation platform("software.amazon.awssdk:bom:2.25.19")
implementation "software.amazon.awssdk:s3"
```

현재 Monitory 프로젝트의 build.gradle에는 S3 클라이언트 의존성이 없고, SQS, SNS, Secrets Manager, IoT Data Plane 등이 포함되어 있었다.

즉 현재 프로젝트는 Presigned URL을 생성하는 구조가 아니라 S3 이벤트를 SQS로 받아 처리하는 구조에 가깝다.

---

## 10. application.yml 설정 예시

Presigned URL을 구현한다면 보통 다음 설정이 필요하다.

```yaml
cloud:
  aws:
    region:
      static: ap-northeast-2
    s3:
      bucket: monitory-equipment-bucket
    credentials:
      access-key: ${AWS_ACCESS_KEY}
      secret-key: ${AWS_SECRET_KEY}
```

각 값의 의미는 다음과 같다.

| 설정                      | 의미                     |
| ----------------------- | ---------------------- |
| cloud.aws.region.static | AWS 리전                 |
| cloud.aws.s3.bucket     | 파일을 저장할 S3 버킷 이름       |
| AWS_ACCESS_KEY          | AWS API 인증용 access key |
| AWS_SECRET_KEY          | AWS API 인증용 secret key |

---

## 11. cloud.aws.region.static

cloud.aws.region.static은 AWS 리전 값을 의미한다.

```yaml
cloud:
  aws:
    region:
      static: ap-northeast-2
```

ap-northeast-2는 AWS 서울 리전이다.

| 리전             | 지역      |
| -------------- | ------- |
| ap-northeast-2 | 서울      |
| us-east-1      | 미국 버지니아 |
| ap-southeast-1 | 싱가포르    |

S3 버킷은 특정 리전에 만들어진다.

예를 들어 S3 버킷이 서울 리전에 있다면 AWS SDK도 서울 리전을 기준으로 요청을 만들어야 한다.

그래서 S3Presigner를 만들 때 region을 지정한다.

```java
.region(Region.of(region))
```

여기서 region 값에 ap-northeast-2가 들어가면 AWS SDK는 서울 리전 기준으로 Presigned URL을 만든다.

---

## 12. S3Config와 @Bean

### 12.1 전체 코드

```java
package com.example.global.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;

@Configuration
public class S3Config {

    @Bean
    public S3Presigner s3Presigner(
            @Value("${cloud.aws.region.static}") String region
    ) {
        return S3Presigner.builder()
                .region(Region.of(region))
                .build();
    }
}
```

---

### 12.2 @Configuration

@Configuration은 설정 클래스라는 뜻이다.

이 클래스는 일반 비즈니스 로직을 처리하기보다는 프로젝트에서 공통으로 사용할 객체를 생성하고 등록하는 역할을 한다.

예를 들어 다음 객체들이 설정 클래스로 등록될 수 있다.

* RestTemplate
* ObjectMapper
* S3Presigner
* SqsAsyncClient
* KafkaTemplate
* ThreadPoolTaskExecutor

---

### 12.3 @Bean을 왜 쓰는가

내가 중간에 질문했던 핵심이 이것이었다.

외부에서 들어오는 객체라서 직접 Bean으로 등록하는 것인지.

정리하면 거의 맞다.

조금 더 정확히 말하면 다음과 같다.

```text
내가 만든 클래스가 아니라 외부 라이브러리 객체이기 때문에
Spring이 자동으로 어떻게 생성해야 할지 모른다.

그래서 개발자가 직접 생성 방법을 알려주고
Spring Bean으로 등록한다.
```

내가 만든 클래스는 다음처럼 등록할 수 있다.

```java
@Service
public class UserService {
}
```

이 경우 Spring은 컴포넌트 스캔으로 UserService를 찾아서 Bean으로 등록한다.

하지만 S3Presigner는 내가 만든 클래스가 아니다.

```text
software.amazon.awssdk.services.s3.presigner.S3Presigner
```

AWS SDK에서 제공하는 외부 라이브러리 객체다.

Spring은 이 객체를 만들 때 어떤 region을 넣어야 하는지, 어떤 인증 방식을 써야 하는지 자동으로 알 수 없다.

그래서 개발자가 직접 알려준다.

```java
@Bean
public S3Presigner s3Presigner() {
    return S3Presigner.builder()
            .region(...)
            .build();
}
```

이 코드는 Spring에게 다음과 같이 말하는 것과 같다.

```text
Spring아, S3Presigner는 이렇게 만들어.
그리고 이 객체를 네가 관리하는 Bean으로 등록해줘.
```

---

### 12.4 @Bean으로 등록하면 뭐가 좋은가

@Bean으로 등록하면 다른 클래스에서 DI로 사용할 수 있다.

```java
@Service
@RequiredArgsConstructor
public class S3PresignedUrlService {

    private final S3Presigner s3Presigner;
}
```

이렇게 작성하면 Spring이 S3Presigner Bean을 자동으로 주입해준다.

즉 매번 직접 new로 만들 필요가 없다.

---

### 12.5 그냥 new로 만들면 안 되나

가능은 하다.

```java
S3Presigner presigner = S3Presigner.create();
```

하지만 실무에서는 보통 이렇게 하지 않는다.

이유는 다음과 같다.

* 객체 생성 로직이 여러 곳에 중복된다.
* region, credentials 같은 설정이 흩어진다.
* 테스트할 때 Mock 객체로 바꾸기 어렵다.
* 객체 생명주기를 Spring이 관리하지 못한다.
* 공통 설정을 한 곳에서 관리하기 어렵다.

그래서 외부 라이브러리 객체는 @Bean으로 등록하고 필요한 곳에서 DI로 주입받는 방식을 많이 사용한다.

---

## 13. PresignedUrlResponse DTO

프론트에게 Presigned URL을 응답하기 위한 DTO다.

```java
package com.example.domain.file.dto;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public class PresignedUrlResponse {

    private String uploadUrl;
    private String key;
}
```

각 필드 의미는 다음과 같다.

| 필드        | 의미                      |
| --------- | ----------------------- |
| uploadUrl | 프론트가 S3에 직접 업로드할 임시 URL |
| key       | S3에 저장될 파일 경로           |

프론트는 uploadUrl로 파일을 업로드하고, key는 나중에 DB에 저장할 때 사용한다.

---

## 14. S3PresignedUrlService

### 14.1 전체 코드

```java
package com.example.domain.file.service;

import com.example.domain.file.dto.PresignedUrlResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.PresignedPutObjectRequest;
import software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest;

import java.time.Duration;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class S3PresignedUrlService {

    private final S3Presigner s3Presigner;

    @Value("${cloud.aws.s3.bucket}")
    private String bucket;

    public PresignedUrlResponse createUploadUrl(String originalFileName, String contentType) {

        String key = "profile/" + UUID.randomUUID() + "-" + originalFileName;

        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType(contentType)
                .build();

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(10))
                .putObjectRequest(putObjectRequest)
                .build();

        PresignedPutObjectRequest presignedRequest =
                s3Presigner.presignPutObject(presignRequest);

        return new PresignedUrlResponse(
                presignedRequest.url().toString(),
                key
        );
    }
}
```

---

## 15. S3PresignedUrlService 코드 설명

### 15.1 S3Presigner 주입

```java
private final S3Presigner s3Presigner;
```

Presigned URL을 생성하는 AWS SDK 객체를 주입받는다.

이 객체는 S3Config에서 @Bean으로 등록했기 때문에 Spring이 자동으로 넣어줄 수 있다.

---

### 15.2 bucket 설정값 주입

```java
@Value("${cloud.aws.s3.bucket}")
private String bucket;
```

application.yml에 있는 S3 버킷 이름을 가져온다.

예시:

```yaml
cloud:
  aws:
    s3:
      bucket: monitory-equipment-bucket
```

그러면 bucket 변수에는 monitory-equipment-bucket 값이 들어간다.

---

### 15.3 key 생성

```java
String key = "profile/" + UUID.randomUUID() + "-" + originalFileName;
```

이 코드는 S3 안에서 파일이 저장될 위치를 만든다.

예를 들어 originalFileName이 profile.png라면 key는 다음처럼 만들어질 수 있다.

```text
profile/550e8400-e29b-41d4-a716-446655440000-profile.png
```

여기서 profile/은 폴더처럼 보이는 prefix이고, UUID는 파일명 중복을 막기 위한 랜덤 고유값이다.

---

### 15.4 UUID란 무엇인가

UUID는 랜덤 고유 ID다.

예시:

```text
550e8400-e29b-41d4-a716-446655440000
```

UUID를 붙이는 이유는 파일명이 겹치는 것을 막기 위해서다.

예를 들어 사용자 100명이 모두 profile.png를 올릴 수 있다.

UUID를 사용하지 않으면 다음처럼 같은 key가 생길 수 있다.

```text
profile/profile.png
```

이 경우 나중에 업로드한 파일이 기존 파일을 덮어쓸 수 있다.

UUID를 붙이면 다음처럼 서로 다른 파일로 저장된다.

```text
profile/550e8400-profile.png
profile/123e4567-profile.png
```

---

### 15.5 PutObjectRequest

```java
PutObjectRequest putObjectRequest = PutObjectRequest.builder()
        .bucket(bucket)
        .key(key)
        .contentType(contentType)
        .build();
```

PutObjectRequest는 S3에 파일을 업로드하겠다는 요청 정보다.

여기에는 다음 정보가 들어간다.

| 값           | 의미               |
| ----------- | ---------------- |
| bucket      | 어느 버킷에 저장할지      |
| key         | 어떤 경로와 이름으로 저장할지 |
| contentType | 파일 타입            |

예시:

```text
bucket = monitory-equipment-bucket
key = profile/550e8400-profile.png
contentType = image/png
```

---

### 15.6 PutObjectPresignRequest

```java
PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
        .signatureDuration(Duration.ofMinutes(10))
        .putObjectRequest(putObjectRequest)
        .build();
```

PutObjectPresignRequest는 방금 만든 업로드 요청에 만료시간을 붙이는 객체다.

이 코드는 다음 뜻이다.

```text
이 S3 업로드 요청을 10분 동안만 사용할 수 있는 URL로 만들어줘.
```

signatureDuration이 10분이면 프론트는 10분 안에 업로드해야 한다.
10분이 지나면 같은 URL로는 업로드할 수 없다.

---

### 15.7 presignPutObject

```java
PresignedPutObjectRequest presignedRequest =
        s3Presigner.presignPutObject(presignRequest);
```

이 부분이 실제 Presigned URL을 생성하는 핵심이다.

AWS SDK는 내부적으로 다음 정보를 사용한다.

* bucket
* key
* contentType
* 만료시간
* access-key
* secret-key
* region
* HTTP method PUT

그리고 AWS 규칙에 맞는 Signature를 만들어 URL에 붙인다.

---

### 15.8 최종 응답

```java
return new PresignedUrlResponse(
        presignedRequest.url().toString(),
        key
);
```

프론트에게 uploadUrl과 key를 반환한다.

응답 예시:

```json
{
  "uploadUrl": "https://monitory-equipment-bucket.s3.ap-northeast-2.amazonaws.com/profile/uuid-profile.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Signature=...",
  "key": "profile/uuid-profile.png"
}
```

---

## 16. FileController

### 16.1 전체 코드

```java
package com.example.domain.file.controller;

import com.example.domain.file.dto.PresignedUrlResponse;
import com.example.domain.file.service.S3PresignedUrlService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequiredArgsConstructor
@RequestMapping("/api/files")
public class FileController {

    private final S3PresignedUrlService s3PresignedUrlService;

    @PostMapping("/presigned-url")
    public ResponseEntity<PresignedUrlResponse> createPresignedUrl(
            @RequestParam String fileName,
            @RequestParam String contentType
    ) {
        PresignedUrlResponse response =
                s3PresignedUrlService.createUploadUrl(fileName, contentType);

        return ResponseEntity.ok(response);
    }
}
```

---

### 16.2 Controller 역할

이 Controller는 파일을 직접 받지 않는다.

프론트가 파일명을 보내면 업로드 가능한 Presigned URL을 만들어서 응답한다.

요청 예시:

```http
POST /api/files/presigned-url?fileName=profile.png&contentType=image/png
```

응답 예시:

```json
{
  "uploadUrl": "https://bucket.s3.ap-northeast-2.amazonaws.com/profile/uuid-profile.png?...",
  "key": "profile/uuid-profile.png"
}
```

---

## 17. 프론트 업로드 코드

프론트는 서버에서 받은 uploadUrl로 S3에 직접 PUT 요청을 보낸다.

```javascript
async function uploadFile(file) {
  const response = await fetch(
    `/api/files/presigned-url?fileName=${file.name}&contentType=${file.type}`,
    {
      method: "POST"
    }
  );

  const data = await response.json();

  await fetch(data.uploadUrl, {
    method: "PUT",
    headers: {
      "Content-Type": file.type
    },
    body: file
  });

  return data.key;
}
```

핵심은 두 번째 fetch다.

```javascript
await fetch(data.uploadUrl, {
  method: "PUT",
  headers: {
    "Content-Type": file.type
  },
  body: file
});
```

이 요청은 Spring 서버가 아니라 S3로 직접 간다.

```text
프론트 → S3
```

---

## 18. URL은 어떻게 만들어지는가

중간에 헷갈렸던 부분은 이것이었다.

```text
AWS한테 URL을 받아오는 게 아니라 내가 직접 만드는 건가?
S3에 직접 올리는데 내가 마음대로 URL을 만드는 게 가능한가?
```

정답은 다음과 같다.

```text
개발자가 직접 만드는 것은 key다.
진짜 인증 URL은 AWS SDK가 만든다.
```

---

### 18.1 개발자가 정하는 것

개발자는 저장 위치를 정한다.

```java
String key = "profile/" + UUID.randomUUID() + "-" + originalFileName;
```

이건 URL이 아니라 S3 내부 파일 경로다.

예시:

```text
profile/550e8400-profile.png
```

---

### 18.2 AWS SDK가 만드는 것

진짜 Presigned URL은 아래 코드에서 만들어진다.

```java
s3Presigner.presignPutObject(presignRequest);
```

이때 AWS SDK가 access-key, secret-key, region, bucket, key, 만료시간을 이용해 Signature를 만든다.

즉 개발자가 문자열을 마음대로 조합해서 인증 URL을 만드는 것이 아니다.

---

## 19. AWS 서버에 요청해서 URL을 받아오는가

Presigned URL 생성은 보통 AWS 서버에 매번 요청해서 받아오는 방식이 아니다.

서버 로컬에서 AWS SDK가 Signature를 계산한다.

이게 가능한 이유는 AWS Signature 생성 규칙이 정해져 있기 때문이다.

AWS SDK는 내 서버 안에서 다음 정보를 이용해 서명을 만든다.

* access-key
* secret-key
* region
* bucket
* key
* 만료시간
* HTTP method

이후 프론트가 Presigned URL로 S3에 요청을 보내면 S3가 그때 검증한다.

S3는 다음을 확인한다.

```text
1. Signature가 올바른가
2. 만료 시간이 지나지 않았는가
3. 이 bucket/key에 대한 권한이 맞는가
4. 요청 method가 맞는가
5. contentType 조건이 맞는가
```

맞으면 업로드를 허용하고, 틀리면 거부한다.

---

## 20. Presigned URL의 실제 모양

Presigned URL은 대략 다음처럼 생겼다.

```text
https://bucket.s3.ap-northeast-2.amazonaws.com/profile/abc.png
?X-Amz-Algorithm=AWS4-HMAC-SHA256
&X-Amz-Date=20260513T120000Z
&X-Amz-Expires=600
&X-Amz-Signature=...
```

각 파라미터 의미는 다음과 같다.

| 파라미터            | 의미        |
| --------------- | --------- |
| X-Amz-Algorithm | 서명 알고리즘   |
| X-Amz-Date      | URL 생성 시각 |
| X-Amz-Expires   | 만료 시간     |
| X-Amz-Signature | AWS 서명값   |

Presigned URL은 단순한 파일 주소가 아니다.

```text
파일 주소 + 임시 인증 정보
```

가 합쳐진 URL이다.

---

## 21. AWS access-key와 secret-key

### 21.1 의미

AWS API를 호출하려면 인증 정보가 필요하다.

```yaml
cloud:
  aws:
    credentials:
      access-key: ${AWS_ACCESS_KEY}
      secret-key: ${AWS_SECRET_KEY}
```

각각의 의미는 다음과 같다.

| 값          | 의미        |
| ---------- | --------- |
| access-key | 아이디 같은 값  |
| secret-key | 비밀번호 같은 값 |

Spring 서버가 S3, SQS, SNS 같은 AWS 서비스를 사용하려면 AWS가 먼저 묻는다.

```text
너 누구야?
```

이때 서버는 access-key와 secret-key로 자신을 인증한다.

---

### 21.2 어디서 발급받는가

AWS IAM에서 발급받는다.

IAM은 AWS 사용자와 권한을 관리하는 서비스다.

발급 흐름은 다음과 같다.

```text
AWS Console
→ IAM
→ User 생성
→ 권한 부여
→ Security credentials
→ Create access key
```

생성하면 다음 두 값이 나온다.

```text
Access Key ID
Secret Access Key
```

Secret Access Key는 생성 시점에만 확인할 수 있다.
잃어버리면 다시 보는 것이 아니라 새로 발급해야 한다.

---

## 22. 실무 보안 관점

### 22.1 yml에 직접 넣으면 위험하다

학습용으로는 아래처럼 직접 넣을 수 있다.

```yaml
cloud:
  aws:
    credentials:
      access-key: AKIA...
      secret-key: abc...
```

하지만 실무에서는 위험하다.

이 파일이 GitHub에 올라가면 AWS 계정 키가 노출된다.

키가 유출되면 다음 문제가 생길 수 있다.

* S3 데이터 유출
* AWS 리소스 무단 생성
* 과금 폭탄
* 크립토 채굴 악용

---

### 22.2 환경변수 사용

그래서 application.yml에는 실제 값을 직접 넣지 않는다.

```yaml
cloud:
  aws:
    credentials:
      access-key: ${AWS_ACCESS_KEY}
      secret-key: ${AWS_SECRET_KEY}
```

그리고 실제 값은 서버 환경변수에 등록한다.

```bash
export AWS_ACCESS_KEY=xxxx
export AWS_SECRET_KEY=yyyy
```

이렇게 하면 코드와 설정 파일에 실제 키가 남지 않는다.

---

### 22.3 IAM Role

EC2 같은 AWS 서버에서 실행한다면 더 좋은 방식은 IAM Role이다.

IAM Role은 서버 자체에 권한을 붙이는 방식이다.

예를 들어 EC2에 다음 권한을 부여할 수 있다.

```text
이 EC2 서버는 특정 S3 버킷에 PutObject 가능
```

그러면 Spring 애플리케이션 코드에 access-key와 secret-key를 직접 넣지 않아도 된다.

AWS SDK가 EC2에 붙은 Role을 자동으로 사용한다.

실무 선호도는 다음과 같다.

| 방식        | 평가                |
| --------- | ----------------- |
| yml 직접 입력 | 위험함               |
| 환경변수      | 개발 환경에서 사용 가능     |
| IAM Role  | AWS 배포 환경에서 가장 권장 |

---

## 23. 현재 Monitory 프로젝트와 Presigned URL의 차이

Cursor가 프로젝트를 분석한 결과, 현재 Monitory 프로젝트에는 Presigned URL을 사용하는 코드는 없었다.

즉 다음과 같은 클래스나 키워드가 없었다.

* S3Presigner
* PutObjectPresignRequest
* GetObjectPresignRequest
* PresignedPutObjectRequest
* software.amazon.awssdk.services.s3.S3Client

현재 프로젝트에서 S3와 관련 있어 보이는 코드는 S3EventSqsListener다.

이 클래스는 Presigned URL을 만드는 클래스가 아니다.

역할은 다음과 같다.

```text
SQS로 들어온 S3 이벤트 메시지를 읽고
어떤 bucket에 어떤 key가 생성됐는지 확인하는 클래스
```

---

## 24. 현재 프로젝트의 S3 이벤트 기반 구조

현재 Monitory 프로젝트의 흐름은 다음과 같다.

```text
장비 예측용 JSON 파일 업로드
→ S3 ObjectCreated 이벤트 발생
→ SQS로 이벤트 전달
→ Spring S3EventSqsListener가 메시지 수신
→ key에서 zone_id, equip_id 추출
→ equipPredictProcessor.equipPredProcess 호출
```

즉 현재 프로젝트에서 중요한 것은 파일 업로드 URL을 발급하는 것이 아니다.

중요한 것은 S3에 파일이 올라왔다는 이벤트를 감지해서 후속 처리를 실행하는 것이다.

---

## 25. S3EventSqsListener 코드 해석

Cursor가 찾은 핵심 코드는 다음과 같다.

```java
for (JsonNode rec : events) {
    String eventName = rec.path("eventName").asText("");
    if (!eventName.startsWith("ObjectCreated")) {
        continue;
    }

    JsonNode s3 = rec.path("s3");
    String bucket = s3.path("bucket").path("name").asText("");
    String keyEnc = s3.path("object").path("key").asText("");
    String key = URLDecoder.decode(keyEnc, StandardCharsets.UTF_8);

    log.info("➡️ bucket={}, key={}", bucket, key);

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
            log.info("👀 서비스 호출 예정 (zoneId={}, equipId={})", zoneId, equipId);
            equipPredictProcessor.equipPredProcess(zoneId, equipId);
        }
    }
}
```

---

### 25.1 ObjectCreated 이벤트만 처리

```java
String eventName = rec.path("eventName").asText("");
if (!eventName.startsWith("ObjectCreated")) {
    continue;
}
```

S3 이벤트에는 파일 생성뿐 아니라 삭제 같은 이벤트도 있을 수 있다.

이 코드는 ObjectCreated로 시작하는 이벤트만 처리한다.

즉 파일이 새로 올라온 경우만 후속 처리를 한다.

---

### 25.2 bucket과 key 읽기

```java
JsonNode s3 = rec.path("s3");
String bucket = s3.path("bucket").path("name").asText("");
String keyEnc = s3.path("object").path("key").asText("");
String key = URLDecoder.decode(keyEnc, StandardCharsets.UTF_8);
```

SQS 메시지 안에 들어 있는 S3 이벤트 JSON에서 bucket 이름과 object key를 꺼낸다.

key는 URL 인코딩되어 있을 수 있기 때문에 URLDecoder로 디코딩한다.

예를 들어 공백이나 한글이 들어가면 인코딩된 형태로 올 수 있다.

---

### 25.3 EQUIPMENT 경로만 처리

```java
if (key.startsWith("EQUIPMENT/") && key.endsWith(".json")) {
```

모든 S3 파일을 처리하는 것이 아니라 EQUIPMENT/로 시작하고 .json으로 끝나는 파일만 처리한다.

즉 장비 예측용 JSON 파일만 대상으로 한다.

---

### 25.4 zone_id와 equip_id 추출

```java
for (String segment : key.split("/")) {
    if (segment.startsWith("zone_id=")) {
        zoneId = segment.substring("zone_id=".length());
    } else if (segment.startsWith("equip_id=")) {
        equipId = segment.substring("equip_id=".length());
    }
}
```

key를 / 기준으로 나눈 뒤 zone_id와 equip_id 값을 찾는다.

예를 들어 key가 다음과 같다고 하자.

```text
EQUIPMENT/zone_id=Z001/equip_id=E001/data.json
```

split 결과는 다음과 같다.

```text
EQUIPMENT
zone_id=Z001
equip_id=E001
data.json
```

이 중 zone_id=로 시작하는 값에서 Z001을 꺼내고, equip_id=로 시작하는 값에서 E001을 꺼낸다.

---

### 25.5 예측 처리 호출

```java
if (zoneId != null && equipId != null) {
    equipPredictProcessor.equipPredProcess(zoneId, equipId);
}
```

zoneId와 equipId가 둘 다 있으면 장비 예측 처리 로직을 호출한다.

즉 이 리스너는 직접 예측을 수행하는 것이 아니라, S3 이벤트를 받아 필요한 식별자를 뽑고 다음 처리 컴포넌트로 넘기는 역할을 한다.

---

## 26. Presigned URL과 현재 프로젝트 구조 비교

| 구분        | Presigned URL     | 현재 Monitory 구조       |
| --------- | ----------------- | -------------------- |
| 목적        | 클라이언트가 S3에 직접 업로드 | S3 업로드 완료 이벤트 처리     |
| 핵심 클래스    | S3Presigner       | S3EventSqsListener   |
| 주요 흐름     | 프론트 → S3          | S3 → SQS → Spring    |
| Spring 역할 | 업로드 URL 발급        | 이벤트 메시지 처리           |
| 사용 상황     | 이미지, 첨부파일, 영상 업로드 | 장비 JSON 업로드 후 예측 트리거 |

---

## 27. Monitory 프로젝트에 Presigned URL을 추가한다면

만약 Monitory 프로젝트에서 외부 시스템이나 프론트가 장비 예측용 JSON을 직접 S3에 올리게 만들고 싶다면 Presigned URL을 사용할 수 있다.

이때 key를 기존 S3EventSqsListener가 이해할 수 있는 형식으로 만들어야 한다.

예시:

```java
String key =
        "EQUIPMENT/zone_id=" + zoneId
        + "/equip_id=" + equipId
        + "/" + UUID.randomUUID() + "-" + fileName;
```

이렇게 만들면 S3에 저장되는 key는 다음처럼 된다.

```text
EQUIPMENT/zone_id=Z001/equip_id=E001/550e8400-data.json
```

그러면 기존 S3EventSqsListener가 key에서 zone_id와 equip_id를 추출할 수 있다.

---

## 28. Monitory용 Presigned URL Service 예시

```java
package com.example.domain.equipment.service;

import com.example.domain.file.dto.PresignedUrlResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.PresignedPutObjectRequest;
import software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest;

import java.time.Duration;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class EquipmentFilePresignedUrlService {

    private final S3Presigner s3Presigner;

    @Value("${cloud.aws.s3.bucket}")
    private String bucket;

    public PresignedUrlResponse createEquipmentJsonUploadUrl(
            String zoneId,
            String equipId,
            String originalFileName
    ) {
        String key =
                "EQUIPMENT/zone_id=" + zoneId
                + "/equip_id=" + equipId
                + "/" + UUID.randomUUID() + "-" + originalFileName;

        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType("application/json")
                .build();

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(10))
                .putObjectRequest(putObjectRequest)
                .build();

        PresignedPutObjectRequest presignedRequest =
                s3Presigner.presignPutObject(presignRequest);

        return new PresignedUrlResponse(
                presignedRequest.url().toString(),
                key
        );
    }
}
```

---

## 29. Monitory용 Controller 예시

```java
package com.example.domain.equipment.controller;

import com.example.domain.equipment.service.EquipmentFilePresignedUrlService;
import com.example.domain.file.dto.PresignedUrlResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequiredArgsConstructor
@RequestMapping("/api/equipment-files")
public class EquipmentFileController {

    private final EquipmentFilePresignedUrlService equipmentFilePresignedUrlService;

    @PostMapping("/presigned-url")
    public ResponseEntity<PresignedUrlResponse> createEquipmentJsonUploadUrl(
            @RequestParam String zoneId,
            @RequestParam String equipId,
            @RequestParam String fileName
    ) {
        PresignedUrlResponse response =
                equipmentFilePresignedUrlService.createEquipmentJsonUploadUrl(
                        zoneId,
                        equipId,
                        fileName
                );

        return ResponseEntity.ok(response);
    }
}
```

요청 예시:

```http
POST /api/equipment-files/presigned-url?zoneId=Z001&equipId=E001&fileName=data.json
```

응답 예시:

```json
{
  "uploadUrl": "https://monitory-equipment-bucket.s3.ap-northeast-2.amazonaws.com/EQUIPMENT/zone_id=Z001/equip_id=E001/uuid-data.json?...",
  "key": "EQUIPMENT/zone_id=Z001/equip_id=E001/uuid-data.json"
}
```

---

## 30. Presigned URL과 S3 이벤트 구조를 함께 쓰는 흐름

Monitory에 Presigned URL을 추가하면 전체 흐름은 다음처럼 연결된다.

```text
1. 외부 시스템 또는 프론트가 Spring에 Presigned URL 요청
   zoneId, equipId, fileName 전달

2. Spring이 key 생성
   EQUIPMENT/zone_id=Z001/equip_id=E001/uuid-data.json

3. Spring이 S3Presigner로 Presigned URL 생성

4. 외부 시스템 또는 프론트가 S3에 직접 PUT 업로드

5. S3에 JSON 파일 생성

6. S3 ObjectCreated 이벤트 발생

7. S3 이벤트가 SQS로 전달

8. Spring의 S3EventSqsListener가 SQS 메시지 수신

9. Listener가 key에서 zone_id, equip_id 추출

10. equipPredictProcessor.equipPredProcess 실행

11. FastAPI 예측 또는 후속 처리 진행

12. 예측 결과 저장 또는 알림 처리
```

즉 Presigned URL은 업로드 입구를 만드는 기술이고, S3 이벤트 구조는 업로드 이후 후속 처리를 자동으로 이어주는 구조다.

둘은 경쟁 관계가 아니라 함께 사용할 수 있다.

---

## 31. 이번 주에 이해한 핵심 정리

### 31.1 기존 업로드 방식

```text
브라우저 → Spring 서버 → S3
```

서버가 파일을 직접 받기 때문에 대용량 파일에서 부하가 커질 수 있다.

---

### 31.2 Presigned URL 방식

```text
브라우저 → Spring 서버
브라우저 → S3
```

Spring 서버는 파일을 받지 않고, S3에 직접 업로드할 수 있는 임시 URL만 발급한다.

---

### 31.3 Bucket과 Key

```text
bucket = 파일 저장소
key = 파일 경로
```

S3에서 key는 폴더처럼 보이지만 실제로는 하나의 문자열이다.

---

### 31.4 URL 생성 주체

개발자는 bucket, key, 만료시간을 정한다.
진짜 인증 URL은 AWS SDK가 secret-key 기반으로 생성한다.

즉 개발자가 마음대로 URL을 조합하는 것이 아니다.

---

### 31.5 UUID

UUID는 파일명 충돌을 막기 위한 랜덤 고유값이다.
같은 파일명을 업로드해도 서로 다른 key가 만들어지도록 한다.

---

### 31.6 @Bean

@Bean은 Spring이 자동으로 만들 수 없는 외부 라이브러리 객체를 직접 생성해서 Spring 컨테이너에 등록하는 방식이다.

S3Presigner는 AWS SDK 객체이므로 @Bean으로 등록해두면 Service에서 DI로 사용할 수 있다.

---

### 31.7 AWS 인증 정보

access-key와 secret-key는 AWS API를 호출하기 위한 인증 키다.
실무에서는 yml에 직접 넣지 않고 환경변수나 IAM Role을 사용한다.

---

### 31.8 현재 Monitory 구조

현재 Monitory 프로젝트는 Presigned URL 발급 구조가 아니라 S3 ObjectCreated 이벤트를 SQS로 받아 후속 처리를 실행하는 구조다.

---

## 32. 이번 주에 배운 점

1. S3는 단순 저장소이고, Bucket은 저장 공간, Key는 파일 경로 역할을 한다.

2. 일반 업로드 방식은 브라우저 → Spring → S3 구조라서 서버 부하가 커질 수 있다.

3. Presigned URL을 사용하면 브라우저 → S3로 직접 업로드할 수 있다.

4. Presigned URL은 단순 URL이 아니라 AWS Signature가 포함된 임시 권한 URL이다.

5. 개발자가 직접 만드는 것은 URL 전체가 아니라 파일 위치인 key다.

6. 진짜 인증 URL은 AWS SDK의 S3Presigner가 만든다.

7. UUID는 파일 이름 중복으로 인한 덮어쓰기를 막기 위해 사용한다.

8. cloud.aws.region.static은 S3가 위치한 AWS 리전을 알려주는 설정이다.

9. @Bean은 외부 라이브러리 객체를 Spring 컨테이너에 직접 등록하기 위해 사용한다.

10. AWS access-key와 secret-key는 IAM에서 발급받는 API 인증 키다.

11. 실무에서는 AWS 키를 코드나 yml에 직접 넣지 않고 환경변수나 IAM Role을 사용한다.

12. 현재 프로젝트는 Presigned URL이 아니라 S3 이벤트 기반 비동기 처리 구조를 사용한다.

13. Presigned URL은 업로드 입구를 만들고, S3 이벤트 구조는 업로드 이후 후속 처리를 자동으로 이어준다.

14. 두 구조는 대체 관계가 아니라 함께 사용할 수 있다.

---

## 33. 다음에 더 보면 좋은 것

* S3EventSqsListener 전체 코드 한 줄씩 분석
* S3 ObjectCreated 이벤트 JSON 구조 분석
* SQS가 S3 이벤트를 어떻게 전달하는지 정리
* EquipPredictProcessor가 S3 이벤트 이후 어떤 처리를 하는지 분석
* Presigned URL 다운로드용 GET URL과 업로드용 PUT URL 차이
* S3 버킷 정책과 IAM 정책의 차이
* S3 CORS 설정이 왜 필요한지
* Presigned URL 만료시간을 어느 정도로 잡는 것이 적절한지