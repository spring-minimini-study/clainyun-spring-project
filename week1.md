# 실시간 센서 모니터링 시스템 클론 코딩 — 진행 로그 & 개념 정리


## 1. 진행한 작업 요약

### 1.1 환경 설정 & 프로젝트 생성

- **IntelliJ에서 Spring Boot 프로젝트 생성**  
  JDK 17, Gradle 선택. → “이 프로젝트는 Java 17로 빌드하고, Gradle로 라이브러리 관리한다”는 뜻.
- **build.gradle에 넣은 것**
  - **Web**: API 서버(컨트롤러/요청-응답 처리)
  - **JPA**: DB 연동(엔티티 ↔ 테이블 매핑)
  - **MySQL**: MySQL 드라이버(실제로 DB랑 통신)
  - **Lombok**: getter/setter/생성자 같은 보일러플레이트(반복적인 ‘템플릿 코드’) 줄임
  - **Security**: 인증·인가(나중에 로그인/권한 처리에 필요)
  - **Actuator**: 헬스 체크(`/actuator/health`) 같은 운영용 엔드포인트  
    → 결론: “서버 띄우고 DB 붙이고 나중에 로그인/헬스체크까지 확장할 준비”를 한 번에 해둔 셋업.
- **yml 3개**  
  `application.yml` = 모든 환경 공통(포트, JPA 옵션 등).  
  `application-local.yml` = 내 PC에서 돌릴 때만 쓰는 DB 주소·계정  
  `application-cloud.yml` = 배포 서버에서 쓸 설정(환경 변수로 DB 주소 받음)  
  → 로컬이랑 배포가 DB가 다르니까 파일을 나눠 둔 거다.
- **패키지**
  - `domain`: Zone, Sensor 같은 **비즈니스 도메인**
  - `global.config`: 전역 설정
  - `global.exception`: 예외 처리
  - `global.security`: 인증/인가
  - `messaging`: Kafka, WebSocket 같은 메시징/이벤트
- **BackendApplication**  
  앱을 실행할 때 제일 먼저 돌아가는 클래스. `main`이 여기 있고, 여기서 Spring이 설정 읽고 DB 연결하고 컨트롤러 올림.
- **DB**  
  처음엔 내 PC에 깔아둔 MariaDB(포트 3306) 썼다가 지우고, 나중에 Docker로 띄운 MySQL(포트 3307)로 바꿈
- **잘 떴는지 확인**  
  Run Configuration에서 **Active profiles**를 `local`로 둔다는 뜻 = “지금은 로컬용 설정(application-local.yml) 쓴다.”  
  서버 띄운 뒤 브라우저나 curl로 **`http://localhost:8080/actuator/health`** 로 접속해서 **`{"status":"UP"}`** 이 나오면, “서버가 떴고 DB 연결까지 됐다”는 뜻이다.  
  (Actuator = Spring이 제공하는 헬스·메트릭용 URL. `/actuator/health`는 “이 앱 살아있어?” 확인용)

### 1.2 Docker MySQL 설정

- **Rancher Desktop 실행 후 `docker context use rancher-desktop`**을 입력함.  
  Rancher로 Docker를 쓰고 있으니까 터미널에서 “지금 docker 명령은 Rancher한테 보낸다”고 알려주는 것.  
  이걸 해야 `docker run` 같은 게 Rancher 쪽 MySQL 컨테이너를 만든다.
- **`docker run`으로 `mysql-monitory-clone` 띄우기**  
  “MySQL 8.0 들어 있는 컨테이너 하나 만들어서 백그라운드로 실행”하는 명령.  
  환경변수(`-e`)로 아래를 “처음 생성 시” 세팅한다.
- **application-local.yml의 url**  
  Spring이 “어느 DB에 붙을지”는 이 url 한 줄로 정한다.  
  `jdbc:mysql://127.0.0.1:3307/monitory_clone` = “내 PC 3307번에 있는 MySQL의 monitory_clone DB 쓴다.”
- **DBeaver**  
  DB 내용 보려고 쓰는 클라이언트.  
  Host 127.0.0.1, Port 3307, DB monitory_clone, user /  로 연결하면 Docker MySQL이 보인다.
- **DBeaver에서 Access denied 나왔을 때**  
  Mac+Rancher에서는 DBeaver가 127.0.0.1로 접속해도 MySQL 입장에선 192.168.65.1로 보인다.  
  MySQL은 “user 이름 + 접속한 IP(host)”로 계정을 구분해서, 그 IP 전용 계정(`user@'192.168.65.1'`)을 컨테이너 안에서 만들어 주고, 인증 방식은 `mysql_native_password`로 맞춰 주면 접속된다.

---

## 2. 개념 정리

### 2.1 build.gradle

**역할**  
이 프로젝트는 Java 몇으로 할지, Spring Boot 몇 버전 쓰고, **어떤 라이브러리** 쓸지를 여기서 정한다.  
Gradle이 이 파일 읽고 필요한 jar 다운받아서 컴파일·실행 가능한 결과물 만든다.

| 구분 | 뜻 / 왜 이렇게 쓰나 |
|------|---------------------|
| **plugins** | `java` = Java 프로젝트로 인식. `spring-boot` = 실행 가능한 jar 만들고 main 찾아서 실행. `dependency-management` = 라이브러리 버전을 Spring Boot에 맞춰 자동으로 맞춰 줌. |
| **implementation** | 실제 서버 돌릴 때 필요한 것.  \n- Web(API)  \n- JPA(DB 매핑)  \n- Security(나중에 로그인)  \n- Actuator(헬스 체크 URL)  \n컴파일할 때도 쓰이고 실행 파일에도 들어감. |
| **compileOnly / annotationProcessor** | “컴파일할 때만” 쓰는 도구. Lombok은 @Getter 같은 걸 보고 getter 메서드 만들어 주는 역할만 해서, 실행 파일에는 안 넣음. |
| **runtimeOnly** | “실행할 때만” 필요. MySQL Connector = MySQL이랑 통신하는 드라이버. 코드에서는 JDBC 인터페이스만 쓰고, 실제로 MySQL 붙을 때 이게 필요함. |
| **testImplementation** | 테스트 코드에서만 쓰는 라이브러리. 본문 빌드에는 안 넣고, 테스트 돌릴 때만 씀. |

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.5.11'
    id 'io.spring.dependency-management' version '1.1.7'
}

group = 'com.monitory'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'com.mysql:mysql-connector-j'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
````

**BackendApplication**

```java
package com.monitory.backend;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BackendApplication {

  public static void main(String[] args) {
    SpringApplication.run(BackendApplication.class, args);
  }
}
```

```
src/main/java/com/monitory/backend/
├── BackendApplication.java
├── domain/
├── global/
│   ├── config/
│   ├── exception/
│   └── security/
└── messaging/
```

### 2.2 application.yml / 프로파일

```yaml
spring:
  config:
    activate:
      on-profile: local
  datasource:
    url: jdbc:mysql://127.0.0.1:3307/monitory_clone
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: user
    password:
  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```

```yaml
spring:
  config:
    activate:
      on-profile: cloud
  datasource:
    url: jdbc:mysql://${RDS_HOST:localhost}:3306/${DB_NAME:monitory_clone}
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: ${RDS_USERNAME:user}
    password: ${RDS_PASSWORD:password}
```

### 2.5 Docker

```bash
docker rm -f mysql-monitory-clone

docker run -d --name mysql-monitory-clone \
  -e MYSQL_ROOT_PASSWORD= \
  -e MYSQL_DATABASE=monitory_clone \
  -e MYSQL_USER=user \
  -e MYSQL_PASSWORD= \
  -p 3307:3306 \
  mysql:8.0

docker ps
```
도커 띄우고 `docker ps`로 도커 잘 돌아가고 있는지 확인하기

```bash
docker exec -it mysql-monitory-clone mysql -uroot -p -e "
CREATE USER 'user'@'192.168.65.1' IDENTIFIED WITH mysql_native_password BY '';
GRANT ALL PRIVILEGES ON monitory_clone.* TO 'user'@'192.168.65.1';
FLUSH PRIVILEGES;
"
```
| 부분                     | 뜻                          |
| ---------------------- | -------------------------- |
| `docker exec`          | **이미 실행중인 컨테이너 안에서 명령 실행** |
| `-it`                  | 터미널처럼 상호작용 가능하게 실행         |
| `mysql-monitory-clone` | 명령을 실행할 **컨테이너 이름**        |
| `mysql`                | 컨테이너 안의 **mysql 클라이언트 실행** |
| `-uroot`               | root 계정으로 로그인              |
| `-p`                   | root **비밀번호 입력 받음**        |
| `-e`                   | 뒤에 있는 **SQL을 바로 실행**       |


**DBeaver에서 Docker DB 보려면**

Host `127.0.0.1`, Port `3307`, Database `monitory_clone`, User `user`, Password `` 로 넣으면 된다.

---

## 3. 로컬 DB 환경

| 항목               | 값                                             | 뜻                                           |
| ---------------- | --------------------------------------------- | ------------------------------------------- |
| DB               | Docker MySQL 8.0 (monitory_clone)             | 지금 쓰는 DB 엔진이랑 DB 이름                         |
| Host             | 127.0.0.1 (또는 localhost)                      | “내 PC” 주소                                   |
| Port             | 3307                                          | Docker MySQL이 열어둔 포트                        |
| DB 이름            | monitory_clone                                | 접속할 데이터베이스 이름                               |
| 계정 (앱 / DBeaver) | user /                                        | Spring이랑 DBeaver 둘 다 이 계정으로 접속              |
| root (관리용)       | root /                                        | 컨테이너 안에서 계정 만들 때 쓰는 관리자 계정                  |
| Spring 프로파일      | local                                         | Run Configuration에서 Active profiles = local |
| DBeaver 접속 시     | Host 127.0.0.1, Port 3307, user /  (또는 root/) |                                             |

---

## 트러블슈팅 정리 — Docker MySQL DBeaver 접속 실패

### 문제

```
Access denied for user 'user'@'192.168.65.1'
```

### 해결

```sql
CREATE USER 'user'@'192.168.65.1'
IDENTIFIED WITH mysql_native_password BY '';

GRANT ALL PRIVILEGES ON monitory_clone.*
TO 'user'@'192.168.65.1';

FLUSH PRIVILEGES;
```

### 배운 점

Docker 환경에서는 **`localhost` ≠ DB가 인식하는 실제 접속 IP(host)**
(특히 Mac + Docker Desktop / Rancher Desktop)

---

DB 접속이 꼬였을 때는 위의 `트러블슈팅 해결 과정 요약` 순서를 그대로 따라가면 된다.
