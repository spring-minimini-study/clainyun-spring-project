# Week6 — 어르신 케어서비스 AI 벡터 임베딩 기반 기관 추천 구조 정리

## 6주차

아래를 중심으로 정리했다.

- 어르신 돌봄 기관 추천 기능이 필요한 이유
- 일반 검색과 AI 추천의 역할
- 벡터 임베딩의 의미
- 의미가 가깝다 / 멀다의 기준
- 코사인 유사도와 추천 점수
- 조건이 여러 개인 문장이 벡터로 표현되는 방식
- Spring 백엔드와 FastAPI AI 서버의 역할
- bge-m3 임베딩 모델과 1024차원 벡터
- bge-m3 모델 선택 이유와 후보군 비교
- PostgreSQL + pgvector 기반 유사도 검색
- 기관 임베딩 생성 / 저장 / 수정 / 삭제 동기화 구조
- 추천 요청 1~8단계 흐름
- AI 서버 전용 RestTemplate 분리 이유
- timeout과 외부 서버 장애 대응
- S3와 Redis의 역할
- AI 추천 결과 후처리와 Presigned URL

---

## 1. 어떤 문제를 해결하려고 했는가

어르신 돌봄 기관을 찾는 과정은 단순히 기관명을 검색하는 문제로 끝나지 않는다.

보호자는 기관을 고를 때 여러 조건을 함께 봐야 한다.

* 어르신의 장기요양등급
* 인지 수준
* 활동 수준
* 거주 지역
* 희망 기관 유형
* 상담 가능 여부
* 예약 가능 여부
* 제공 서비스
* 전문 질환 대응 가능 여부
* 시설 특징
* 운영 특징

예를 들어 보호자가 원하는 상황은 다음과 같을 수 있다.

```text
인지 저하가 있는 어르신이고,
낮 시간 동안 돌봄이 필요하며,
식사 지원과 인지 활동 프로그램이 있는 기관을 찾고 싶다.
````

이 경우 사용자는 기관 목록을 하나씩 보면서 아래 과정을 직접 해야 한다.

```text
지역 확인
→ 기관 유형 확인
→ 장기요양등급 대응 여부 확인
→ 기관 설명 확인
→ 상담 가능 여부 확인
→ 예약 가능 여부 확인
```

이 과정이 길어지면 보호자는 어떤 기관이 어르신에게 맞는지 판단하기 어려워진다.

그래서 어르신 케어서비스에서는 사용자가 모든 기관을 직접 비교하지 않아도 되도록, 사용자 조건과 기관 정보를 바탕으로 적합한 후보를 먼저 제안하는 추천 기능을 설계했다.

핵심:

* 돌봄 기관 탐색은 조건이 많다
* 보호자가 모든 기관을 직접 비교하기 어렵다
* 추천 기능은 사용자 상황에 맞는 후보를 먼저 보여주는 역할을 한다

---

## 2. 일반 검색과 AI 추천의 역할

기관 탐색에는 일반 검색과 AI 추천이 함께 사용될 수 있다.

### 2-1. 일반 검색

일반 검색은 사용자가 명확한 조건을 넣고 기관을 찾는 방식이다.

예:

```text
지역 = 서울
기관 유형 = 주야간보호센터
상담 가능 = true
```

이 경우 백엔드는 조건에 맞는 기관을 조회한다.

```text
조건 입력
→ 조건에 맞는 기관 조회
→ 기관 목록 반환
```

핵심:

* 일반 검색은 정형 조건을 기준으로 기관을 찾는다
* 지역, 기관 유형, 상담 가능 여부처럼 값이 명확한 조건에 적합하다

---

### 2-2. AI 추천

AI 추천은 사용자의 상황과 기관 정보를 의미적으로 연결하는 방식이다.

예:

```text
사용자 조건:
인지 저하가 있고 낮 시간 돌봄이 필요한 어르신

기관 설명:
주야간보호센터로 인지 활동 프로그램과 식사 지원을 제공합니다.
```

두 문장이 같은 단어를 그대로 쓰지 않아도 의미가 연결되면 추천 후보가 될 수 있다.

핵심:

* AI 추천은 문장과 조건의 의미를 반영한다
* 사용자가 정확한 기관 유형이나 서비스명을 몰라도 후보를 받을 수 있다
* 일반 검색은 조건을 직접 지정할 때, AI 추천은 상황에 맞는 후보를 제안할 때 유용하다

---

## 3. 검색과 추천을 함께 둔 이유

검색과 추천은 둘 중 하나만 선택하는 관계가 아니다.

사용자가 이미 원하는 조건을 알고 있을 때는 검색이 편하다.

예:

```text
서울 강남구에 있는 주야간보호센터
```

이 경우에는 지역, 기관 유형, 거리 같은 정형 조건으로 빠르게 조회하면 된다.

반대로 사용자가 정확한 조건명을 모를 때는 추천이 더 자연스럽다.

예:

```text
인지 저하가 있고 낮 시간 동안 돌봄이 필요한 어르신에게 맞는 기관
```

이런 문장은 단순 필터만으로 처리하기 어렵다.
사용자의 상황과 기관 설명의 의미를 비교해야 한다.

그래서 어르신 케어서비스에서는 일반 검색과 AI 추천을 함께 가져가는 구조가 적합하다.

핵심:

* 검색은 사용자가 조건을 알고 있을 때 유용하다
* 추천은 사용자가 상황만 설명할 때 유용하다
* 두 기능이 함께 있어야 기관 탐색 경험이 자연스러워진다

---

## 4. 벡터 임베딩이란 무엇인가

벡터 임베딩은 문장, 단어, 기관 설명 같은 텍스트를 컴퓨터가 계산할 수 있는 숫자 배열로 바꾸는 것이다.

사람이 보는 문장:

```text
인지 저하 어르신을 위한 낮 시간 돌봄이 필요합니다.
```

AI 모델이 변환한 벡터 예시:

```text
[0.12, -0.43, 0.88, 0.05, ...]
```

이 숫자 배열을 벡터라고 부른다.

핵심:

* 문장 → 숫자 배열
* 숫자 배열 → 벡터
* 문장을 벡터로 바꾸는 과정 → 임베딩

---

## 5. 문장을 숫자로 바꾸는 이유

컴퓨터는 문장의 의미를 사람처럼 바로 이해하지 못한다.

예를 들어 아래 두 문장을 보자.

```text
인지 저하 어르신을 위한 낮 시간 돌봄이 필요합니다.
```

```text
치매 예방 프로그램과 주야간보호 서비스를 제공합니다.
```

두 문장은 단어가 완전히 같지는 않지만 의미상 연결된다.

사람은 이 관계를 바로 이해할 수 있지만, 컴퓨터는 단순 문자열만 보고 두 문장이 비슷한지 판단하기 어렵다.

그래서 문장을 숫자 벡터로 바꾼 뒤 벡터끼리 계산한다.

```text
문장 A → [0.1, 0.5, 0.2, ...]
문장 B → [0.1, 0.4, 0.3, ...]
문장 C → [0.9, -0.2, 0.1, ...]
```

A와 B의 숫자 배열이 가까우면 의미도 비슷하다고 볼 수 있다.

핵심:

* 문장을 숫자로 바꾸면 계산할 수 있다
* 벡터가 가까우면 의미도 비슷하다고 판단할 수 있다
* 추천은 이 벡터 간 가까움을 이용한다

---

## 6. 의미가 가깝다 / 멀다는 기준

의미가 가깝다는 말은 감으로 판단하는 것이 아니다.

임베딩 추천에서는 두 벡터 사이의 거리나 방향의 유사도를 기준으로 판단한다.

예:

```text
문장 A: 인지 저하 어르신을 위한 낮 시간 돌봄이 필요합니다.
→ [0.12, 0.83, -0.21, ...]

문장 B: 치매 예방 프로그램과 주야간보호 서비스를 제공합니다.
→ [0.10, 0.79, -0.18, ...]

문장 C: 재활 운동과 물리치료 중심 서비스를 제공합니다.
→ [-0.55, 0.11, 0.68, ...]
```

A와 B는 벡터 방향이 비슷하고, A와 C는 방향이 덜 비슷하다.

이때 모델은 A와 B가 의미적으로 더 연결된다고 판단한다.

핵심:

* 의미의 가까움 = 임베딩 벡터의 유사도
* 벡터가 비슷하면 모델이 보기에는 문맥이나 의도가 연결된다

---

## 7. 코사인 유사도란 무엇인가

벡터 유사도를 비교할 때 자주 쓰는 기준 중 하나가 코사인 유사도다.

코사인 유사도는 두 벡터의 방향이 얼마나 비슷한지를 본다.

```text
같은 방향      → 1에 가까움
직각에 가까움   → 0에 가까움
반대 방향      → -1에 가까움
```

화살표로 보면 다음과 같다.

```text
↗  ↗  → 비슷한 방향 = 유사도 높음
↗  ↙  → 반대 방향 = 유사도 낮음
```

핵심:

* 코사인 유사도는 벡터의 방향을 비교한다
* 방향이 비슷할수록 의미가 가깝다고 판단한다

---

## 8. 유사도는 1을 넘을 수 있는가

코사인 유사도 기준이면 유사도 값은 보통 아래 범위 안에 있다.

```text
-1 ~ 1
```

문장 임베딩 추천에서는 보통 이렇게 해석한다.

```text
0에 가까움 → 관련 적음
1에 가까움 → 매우 유사함
```

예:

```text
0.91 → 의미가 매우 가까움
0.63 → 어느 정도 관련 있음
0.21 → 관련이 약함
```

따라서 코사인 유사도 자체는 1을 넘지 않는다.

다만 서비스에서 보여줄 때는 0.87을 87점처럼 변환할 수 있다.

```text
0.87 × 100 = 87점
```

핵심:

* 코사인 유사도 자체는 보통 1을 넘지 않는다
* 화면이나 API 응답에서는 0~100점 형태로 변환할 수 있다

---

## 9. 조건이 여러 개인 문장은 벡터로 어떻게 표현되는가

사용자가 한 문장 안에 여러 조건을 넣을 수 있다.

예:

```text
서울에 거주하고,
인지 저하가 있고,
낮 시간 돌봄이 필요하고,
식사 지원이 가능하고,
상담 예약이 가능한 주야간보호센터를 찾고 싶다.
```

조건이 여러 개여도 AI 모델은 이 문장 전체를 하나의 벡터로 바꾼다.

설명할 때는 벡터를 3차원처럼 단순화해서 보여주기도 하지만, 실제 임베딩 벡터는 훨씬 길다.

예:

```text
[0.12, -0.43, 0.88, 0.05, 0.77, -0.31, ...]
```

어르신 케어서비스의 AI 서버에서는 bge-m3 모델을 사용하며, 코드상 1024차원 임베딩을 생성하는 구조다.

즉 세 방향만 있는 것이 아니라 수많은 의미 축을 가진 공간에 문장을 배치한다고 보면 된다.

핵심:

* 조건이 여러 개여도 문장 전체가 하나의 고차원 벡터로 표현된다
* 벡터 안에는 지역, 인지 상태, 희망 서비스, 기관 유형 같은 의미가 함께 반영된다
* 각 차원이 정확히 어떤 조건 하나를 뜻한다고 사람이 직접 해석하기는 어렵다

---

## 10. 조건이 많을 때 추천을 구성하는 방식

조건이 많다고 항상 추천이 좋아지는 것은 아니다.

예:

```text
서울에 있고,
인지 프로그램이 있고,
재활 치료도 가능하고,
24시간 입소도 가능하고,
상담 예약도 가능하고,
식사 지원도 가능하고,
가격도 저렴하고,
집에서 10분 거리이고,
여성 어르신 전용이고,
주말 운영도 되는 기관
```

이렇게 조건이 많으면 모델은 전체 의미를 하나의 벡터로 만들지만, 각 조건의 중요도를 완벽하게 반영한다고 보장할 수는 없다.

그래서 추천 시스템에서는 보통 아래처럼 나누는 방식이 적합하다.

```text
정형 조건 → DB 필터
문장형 조건 → 임베딩 유사도
```

예:

```text
지역 = 서울              → DB 필터
상담 가능 = true         → DB 필터
기관 유형 = 주야간보호센터 → DB 필터
상세 요구사항 문장        → 임베딩 유사도
```

핵심:

* 명확한 조건은 필터로 먼저 줄인다
* 의미 비교가 필요한 부분을 임베딩 유사도로 처리한다
* 정형 조건과 문장형 조건을 나누면 추천 흐름이 더 안정적이다

---

## 11. AI 서버 위치와 역할

어르신 케어서비스에서 AI 서버는 Spring 프로젝트 내부 클래스가 아니라 별도로 실행되는 FastAPI 서버다.

구조는 다음과 같다.

```text
사용자
→ Spring Boot 백엔드
→ FastAPI AI 서버
→ 임베딩 생성 / 저장 / 추천 검색
```

AI 서버 코드에는 FastAPI 앱이 생성되어 있고, 추천과 임베딩 관련 API가 정의되어 있다.

핵심:

* Spring 백엔드 안에서 AI 모델을 직접 실행하는 구조가 아니다
* AI 모델과 벡터 검색은 별도 FastAPI 서버가 담당한다
* Spring은 AI 서버를 호출하고 결과를 서비스 응답으로 조립한다

---

## 12. FastAPI가 사용된 이유

FastAPI는 Python 기반 API 서버 프레임워크다.

AI 모델, 임베딩 모델, 벡터 유사도 계산은 Python 생태계에서 구현하는 경우가 많다.
따라서 AI 기능을 별도 서버로 분리할 때 FastAPI를 사용하면 모델 로딩, 추론, API 응답 처리를 하나의 Python 서버 안에서 구성할 수 있다.

역할을 나누면 다음과 같다.

| 구분              | 역할                                        |
| --------------- | ----------------------------------------- |
| Spring Boot 백엔드 | 사용자 인증, 프로필 조회, 기관 DB 조회, AI 서버 호출, 응답 조립 |
| FastAPI AI 서버   | 임베딩 생성, 벡터 저장, 유사도 계산, 추천 후보 반환           |

핵심:

* Spring = 서비스 흐름 담당
* FastAPI = AI 계산 담당
* 역할을 나누면 유지보수와 확장이 쉬워진다

---

## 13. 실제 AI 서버 엔드포인트

AI 서버에는 추천과 임베딩 관련 엔드포인트가 구현되어 있다.

주요 엔드포인트는 다음과 같다.

* `POST /api/v1/institutions/embeddings`
* `POST /api/v1/users/profile-text`
* `POST /api/v1/users/profile-embedding`
* `POST /api/v1/recommendations`

핵심:

* 기관 임베딩 생성/저장 API가 있다
* 사용자 프로필 텍스트/임베딩 생성 API가 있다
* 추천 API가 있다
* 추천 계산은 FastAPI 서버에서 수행된다

---

## 14. bge-m3 임베딩 모델

AI 서버에서는 `BAAI/bge-m3` 모델을 사용해 텍스트를 임베딩 벡터로 변환한다.

구조는 다음과 같다.

```python
class EmbeddingService:
    """bge-m3 임베딩 생성 서비스"""
    
    def __init__(self):
        self.model = SentenceTransformer('BAAI/bge-m3')
    
    def encode_text(self, text: str) -> np.ndarray:
        embedding = self.model.encode(text, normalize_embeddings=True)
        return embedding
```

이 코드는 텍스트를 입력받아 bge-m3 모델로 임베딩 벡터를 생성하는 역할을 한다.

핵심:

* bge-m3 = 텍스트를 벡터로 바꾸는 임베딩 모델
* 사용자 조건과 기관 정보를 같은 벡터 공간에 올려놓기 위해 사용한다
* 같은 공간에 있어야 유사도 비교가 가능하다

---

## 15. bge-m3 모델을 선택한 이유

임베딩 모델을 고를 때 중요한 기준은 단순히 모델 이름이 아니라, 서비스 데이터와 잘 맞는지다.

어르신 케어서비스의 추천 데이터는 대부분 한국어 문장이다.

예:

```text
인지 저하 어르신을 위한 주야간보호
치매 예방 프로그램
식사 지원
장기요양등급
기관 운영 특징
시설 특징
```

이런 문장을 잘 벡터화하려면 한국어 문장 의미를 잘 반영하는 임베딩 모델이 필요하다.

bge-m3를 선택한 이유는 다음과 같이 정리할 수 있다.

* 한국어를 포함한 다국어 문장 임베딩에 활용 가능하다
* 문장 간 의미 유사도 비교에 적합하다
* 검색 / 추천 / RAG 같은 retrieval 작업에 많이 쓰이는 계열이다
* 1024차원 벡터를 생성해 문장의 의미 정보를 풍부하게 담을 수 있다
* `SentenceTransformer`로 로드해서 FastAPI 서버 안에서 직접 사용할 수 있다
* 외부 API 호출 없이 자체 AI 서버에서 임베딩을 생성할 수 있다

핵심:

* 추천 기능의 핵심은 문장 의미 비교다
* bge-m3는 기관 설명과 사용자 조건을 같은 벡터 공간에 올려놓기 적합하다
* 자체 서버에서 임베딩을 생성할 수 있어 AI 서버 구조와 잘 맞는다

---

## 16. 임베딩 모델 후보군

bge-m3 외에도 임베딩 모델 후보는 여러 가지가 있을 수 있다.

### 16-1. OpenAI Embedding API

OpenAI의 `text-embedding` 계열 모델을 사용하는 방식이다.

장점:

* 임베딩 품질이 좋다
* 별도 모델 서버를 직접 운영하지 않아도 된다
* API 호출만으로 쉽게 사용할 수 있다

고려할 점:

* 외부 API 비용이 발생한다
* 네트워크 의존성이 생긴다
* 사용자/기관 데이터가 외부 API로 전송된다
* 서비스 내부에서 모델을 직접 제어하기 어렵다

---

### 16-2. Ko-SBERT / KR-SBERT 계열

한국어 문장 임베딩에 자주 사용되는 모델 계열이다.

장점:

* 한국어 문장 유사도에 적합하다
* 비교적 가볍게 사용할 수 있다
* 한국어 중심 서비스와 잘 맞는다

고려할 점:

* 모델별 성능 차이가 있다
* 최신 다국어 retrieval 모델과 성능을 비교해봐야 한다
* 기관 추천처럼 다양한 태그와 설명을 포함하는 데이터에서 별도 검증이 필요하다

---

### 16-3. multilingual-e5 계열

다국어 검색과 문장 유사도에 사용되는 임베딩 모델 계열이다.

장점:

* 다국어 검색에 강하다
* retrieval 작업에 많이 사용된다
* 사용자 질의와 문서 검색 구조에 적합하다

고려할 점:

* 입력 문장에 `query:` / `passage:` 같은 prefix 사용 방식이 권장되는 경우가 있다
* 서비스 입력 포맷을 모델 특성에 맞게 관리해야 한다
* 모델 크기에 따라 서버 자원 부담이 달라진다

---

### 16-4. all-MiniLM 계열

가볍고 빠른 문장 임베딩 모델 계열이다.

장점:

* 빠르다
* 서버 자원 부담이 작다
* 간단한 문장 유사도 실험에 적합하다

고려할 점:

* 한국어 서비스에서는 의미 표현력이 부족할 수 있다
* 돌봄 기관 추천처럼 도메인 문맥이 중요한 경우에는 품질 검증이 필요하다

---

### 16-5. 최종 선택 기준

모델 선택 기준은 다음과 같이 정리할 수 있다.

| 기준           | 의미                              |
| ------------ | ------------------------------- |
| 한국어 처리       | 어르신 케어서비스 데이터가 대부분 한국어이기 때문     |
| 문장 의미 비교     | 추천은 단어 일치보다 의미 유사도가 중요하기 때문     |
| 자체 서버 운영 가능성 | FastAPI AI 서버에서 직접 임베딩을 생성하기 때문 |
| 검색/추천 적합성    | 기관 벡터와 사용자 벡터를 비교해야 하기 때문       |
| 비용과 의존성      | 외부 API 비용과 네트워크 의존성을 줄이기 위함     |
| 확장성          | 기관 수와 추천 요청 증가를 고려해야 하기 때문      |

핵심:

* 후보군은 OpenAI Embedding API, Ko-SBERT, multilingual-e5, all-MiniLM 등이 있을 수 있다
* 어르신 케어서비스에서는 한국어 의미 비교와 자체 서버 운영 관점에서 bge-m3가 적합하다
* 모델 선택은 정확도만이 아니라 비용, 운영, 데이터 보안, 서버 구조까지 함께 고려해야 한다

---

## 17. 기관 임베딩 생성 / 저장 흐름

기관 정보가 등록되거나 동기화될 때 AI 서버는 기관 정보를 추천에 사용할 수 있는 형태로 변환한다.

흐름:

```text
기관 정보 수신
→ 기관 정보를 텍스트로 변환
→ bge-m3로 임베딩 생성
→ metadata 구성
→ DB에 저장
```

기관 정보 예:

```text
기관명: 햇살 주야간보호센터
기관 유형: 주야간보호
주소: 서울시 강남구
전문 질환: 치매, 인지 저하
서비스 유형: 식사 지원, 인지 활동
시설 특징: 프로그램실, 휴게 공간
```

AI 서버는 이런 정보를 하나의 추천용 텍스트로 정리한 뒤 임베딩한다.

핵심:

* 기관 정보를 추천용 텍스트로 정규화한다
* 정규화된 텍스트를 벡터화한다
* 벡터와 metadata를 함께 저장해 추천 응답에 활용한다

---

## 18. 기관 등록 / 수정 / 삭제와 임베딩 동기화

AI 추천에서 중요한 점은 서비스 DB의 기관 정보와 AI 서버의 기관 임베딩 정보가 함께 맞아야 한다는 것이다.

기관이 새로 등록되었는데 AI 서버에 임베딩이 없으면 추천 결과에 반영되기 어렵다.

```text
DB에는 새 기관 있음
AI 서버에는 해당 기관 임베딩 없음
```

기관 정보가 수정되었는데 AI 서버의 임베딩이 예전 정보라면 추천 결과가 최신 기관 정보를 반영하지 못한다.

```text
DB 기관 설명은 수정됨
AI 서버 임베딩은 예전 설명 기준
```

기관이 삭제되었는데 AI 서버에 임베딩이 남아 있으면 더 이상 유효하지 않은 기관이 추천될 수 있다.

```text
DB에서는 기관 삭제
AI 서버에는 기관 벡터 남아 있음
```

그래서 기관 데이터 변경 시 AI 서버의 임베딩도 함께 갱신되어야 한다.

```text
기관 등록 → AI 서버에 임베딩 생성 요청
기관 수정 → AI 서버에 임베딩 갱신 요청
기관 삭제 → AI 서버에 임베딩 삭제 요청
```

핵심:

* 추천에 쓰이는 벡터 데이터는 실제 기관 데이터와 일치해야 한다
* 기관 등록/수정/삭제는 AI 서버 임베딩 동기화와 연결되어야 한다
* 임베딩 동기화는 추천 품질뿐 아니라 데이터 정합성과도 관련된다

---

## 19. pgvector란 무엇인가

pgvector는 PostgreSQL에서 벡터 데이터를 저장하고 검색할 수 있게 해주는 확장이다.

일반 PostgreSQL은 숫자, 문자열, 날짜 같은 데이터를 다룬다.
pgvector를 사용하면 아래와 같은 벡터도 저장하고 검색할 수 있다.

```text
[0.12, -0.43, 0.88, ...]
```

이 프로젝트에서는 기관 임베딩을 PostgreSQL + pgvector로 저장하고, 사용자 조건 벡터와 가까운 기관 벡터를 검색하는 구조를 사용한다.

핵심:

* PostgreSQL = 기본 DB
* pgvector = 벡터 저장 / 유사도 검색 기능 추가
* 기관 추천에서는 벡터 검색을 위해 필요하다

---

## 20. pgvector 유사도 검색 코드 의미

AI 서버의 추천 검색은 다음 형태의 SQL로 수행된다.

```sql
SELECT 
    institution_id,
    1 - (embedding <=> %s::vector) AS similarity,
    metadata,
    original_text
FROM institution_embeddings
WHERE 1 - (embedding <=> %s::vector) >= %s
ORDER BY embedding <=> %s::vector
LIMIT %s
```

의미:

* `embedding` = 기관 벡터
* `%s::vector` = 사용자 조건 벡터
* `<=>` = pgvector의 벡터 거리 연산
* `1 - distance` = 유사도 점수
* `ORDER BY embedding <=> ...` = 가까운 벡터부터 정렬
* `LIMIT` = 상위 N개만 반환

핵심:

* 거리가 가까울수록 의미가 가깝다
* 거리가 가까우면 similarity 값이 높다
* 추천 결과는 유사도가 높은 기관 순서로 반환된다

---

## 21. 추천 요청 DTO 구조

FastAPI AI 서버는 Spring 백엔드가 보내는 추천 요청 형식을 모델로 정의한다.

구조는 다음과 같다.

```python
class RecommendationRequest(BaseModel):
    """추천 요청 (Spring에서 전달)"""
    member: MemberInfo = Field(..., description="회원 정보")
    elderly: ElderlyInfo = Field(..., description="어르신 프로필 정보")
    additionalText: Optional[str] = Field(default="", description="추가 요구사항")
    limit: int = Field(default=5, description="추천 기관 수")
```

핵심:

* member = 보호자/회원 정보
* elderly = 어르신 프로필 정보
* additionalText = 추가 요구사항
* limit = 추천받을 기관 수

---

## 22. 사용자 프로필 텍스트를 만드는 이유

AI 서버는 `member`, `elderly`, `additionalText`를 각각 따로 비교하는 것이 아니라, 추천에 사용할 하나의 텍스트로 정리한다.

예:

```text
보호자 선호 조건:
인지 프로그램, 식사 지원, 주야간보호

어르신 상태:
인지 저하가 있고 낮 시간 돌봄이 필요함

추가 요청:
집에서 가까운 곳이면 좋겠음
```

이 정보들을 하나의 추천용 문장으로 정리하면 다음과 같은 형태가 된다.

```text
인지 저하가 있는 어르신이며 낮 시간 돌봄이 필요합니다.
보호자는 인지 프로그램, 식사 지원, 주야간보호 서비스를 선호합니다.
집에서 가까운 기관을 원합니다.
```

이렇게 하나의 텍스트로 정리해야 bge-m3 모델이 사용자 조건을 하나의 벡터로 변환할 수 있다.

핵심:

* 추천 요청에 필요한 정보는 여러 객체에 흩어져 있다
* AI 서버는 이 정보를 하나의 추천용 텍스트로 정규화한다
* 정규화된 텍스트를 임베딩해야 기관 벡터와 비교할 수 있다

---

## 23. 추천 API 내부 흐름

추천 엔드포인트는 `POST /api/v1/recommendations` 형태로 구현되어 있다.

흐름은 다음과 같다.

```text
1. member 정보와 elderly 정보 수신
2. 사용자 프로필 텍스트 생성
3. bge-m3로 사용자 텍스트 임베딩 생성
4. pgvector로 유사 기관 검색
5. metadata에서 기관 정보와 태그 조립
6. RecommendationItem 생성
7. RecommendationResponse 반환
```

핵심:

* 사용자의 상태와 선호 조건도 하나의 텍스트로 만든다
* 그 텍스트를 벡터로 변환한다
* 기관 벡터들과 비교해 유사도가 높은 기관을 추천한다

---

## 24. 추천 결과에 포함되는 정보

AI 서버는 추천 기관을 `RecommendationItem` 형태로 반환한다.

포함되는 정보 예:

* institutionId
* similarity
* name
* type
* address
* isAvailable
* tags
* recommendationReason

예시:

```json
{
  "institutionId": 12,
  "similarity": 0.91,
  "name": "햇살 주야간보호센터",
  "type": "주야간보호",
  "address": "서울시 강남구 ...",
  "isAvailable": true,
  "tags": ["치매", "인지 활동", "식사 지원"],
  "recommendationReason": "유사도 91.00%로 매칭되었습니다."
}
```

핵심:

* AI 서버는 유사도와 기관 메타데이터를 조합해 추천 후보를 반환한다
* 화면에 필요한 추가 정보는 Spring 백엔드에서 후처리할 수 있다

---

## 25. 추천 요청 1~8단계 흐름

추천 흐름은 다음과 같다.

```text
1. 사용자가 추천 요청
2. 백엔드가 사용자 정보와 어르신 프로필 조회
3. 백엔드가 RecommendationRequest 생성
4. AI 서버에 추천 요청
5. AI 서버가 유사도 기반 기관 후보 반환
6. 백엔드가 기관 상세 정보 조회
7. 이미지 URL, 상담 가능 정보 등 후처리
8. 클라이언트에 추천 결과 응답
```

이 흐름을 계층으로 보면 다음과 같다.

```text
Controller
→ RecommendationService
→ AiServerClient
→ FastAPI AI Server
→ pgvector
→ RecommendationService 후처리
→ Response DTO 반환
```

핵심:

* Controller는 요청을 받는다
* Service는 추천 흐름을 조립한다
* AI 서버는 유사도 계산을 담당한다
* 백엔드는 최종 응답 형태로 다시 정리한다

---

## 26. 1단계: 사용자가 추천 요청

클라이언트는 추천 API를 호출한다.

예시 요청:

```json
{
  "elderlyProfileId": 3,
  "additionalRequest": "인지 저하가 있고 낮 시간 돌봄이 가능한 기관을 찾고 싶어요."
}
```

Spring Controller 예시:

```java
@RestController
@RequestMapping("/api/recommendations")
@RequiredArgsConstructor
public class RecommendationController {

    private final RecommendationService recommendationService;

    @PostMapping
    public List<RecommendationResponse> recommend(
            @AuthenticationPrincipal CustomUserDetails userDetails,
            @RequestBody RecommendationApiRequest request
    ) {
        Long memberId = userDetails.getMemberId();
        return recommendationService.recommend(memberId, request);
    }
}
```

핵심:

* 클라이언트는 추천 조건을 보낸다
* 백엔드는 로그인 사용자와 요청 내용을 기반으로 추천 흐름을 시작한다

---

## 27. 2단계: 백엔드가 사용자 정보와 어르신 프로필 조회

AI 추천을 요청하려면 사용자와 어르신 프로필 정보가 필요하다.

예시 코드:

```java
Member member = memberRepository.findById(memberId)
        .orElseThrow(() -> new NotFoundException("사용자를 찾을 수 없습니다."));

ElderlyProfile profile = elderlyProfileRepository.findById(apiRequest.getElderlyProfileId())
        .orElseThrow(() -> new NotFoundException("어르신 프로필을 찾을 수 없습니다."));
```

조회하는 이유:

* 사용자 선호 조건 확인
* 어르신 상태 확인
* AI 서버로 보낼 요청 데이터 구성

핵심:

* 추천 요청은 단순 문자열 하나가 아니라 사용자/어르신 프로필 기반으로 만들어진다

---

## 28. 3단계: RecommendationRequest 생성

백엔드는 조회한 사용자/어르신 정보를 AI 서버가 이해할 수 있는 요청 DTO로 만든다.

예시 코드:

```java
private RecommendationRequest buildRecommendationRequest(
        Long memberId,
        ElderlyProfile profile,
        RecommendationApiRequest apiRequest
) {
    String requestText =
            "장기요양등급: " + profile.getCareGrade() + ", "
          + "인지 상태: " + profile.getCognitiveStatus() + ", "
          + "희망 지역: " + profile.getRegion() + ", "
          + "희망 기관 유형: " + profile.getPreferredInstitutionType() + ", "
          + "추가 요청: " + apiRequest.getAdditionalRequest();

    return new RecommendationRequest(
            memberId,
            profile.getId(),
            profile.getCareGrade(),
            profile.getCognitiveStatus(),
            profile.getRegion(),
            profile.getPreferredInstitutionType(),
            requestText
    );
}
```

핵심:

* DB에 흩어진 정보를 AI 서버 요청 형식으로 조립한다
* 정형 정보와 문장형 요청을 함께 담는다

---

## 29. 4단계: AI 서버에 추천 요청

Spring 백엔드는 AI 서버에 HTTP 요청을 보낸다.

예시 코드:

```java
@Component
@RequiredArgsConstructor
public class AiServerClient {

    @Qualifier("aiServerRestTemplate")
    private final RestTemplate aiServerRestTemplate;

    @Value("${ai.server.base-url}")
    private String aiServerBaseUrl;

    public List<AiRecommendationItem> recommend(RecommendationRequest request) {
        String url = aiServerBaseUrl + "/api/v1/recommendations";

        ResponseEntity<AiRecommendationItem[]> response =
                aiServerRestTemplate.postForEntity(
                        url,
                        request,
                        AiRecommendationItem[].class
                );

        AiRecommendationItem[] body = response.getBody();

        if (body == null) {
            return List.of();
        }

        return Arrays.asList(body);
    }
}
```

핵심:

* Spring은 직접 유사도 계산을 하지 않는다
* AI 서버에 추천 계산을 맡긴다
* Spring은 요청을 만들고 결과를 받는다

---

## 30. AI 서버 전용 RestTemplate을 분리하는 이유

백엔드는 AI 서버 외에도 여러 외부 API를 호출할 수 있다.

예:

* Kakao 주소 변환 API
* S3 파일 관련 API
* AI 추천 서버 API

그런데 모든 외부 API 호출을 같은 설정으로 처리하면 운영 중 문제가 생길 수 있다.

AI 서버는 임베딩 생성이나 벡터 검색 때문에 응답 시간이 길어질 수 있다.
주소 변환 API는 비교적 빠른 응답이 필요할 수 있다.

그래서 AI 서버 호출은 전용 `RestTemplate`으로 분리하는 것이 좋다.

```text
aiServerRestTemplate
→ AI 서버 전용 timeout
→ AI 서버 전용 error handler
→ AI 서버 base-url 관리
```

핵심:

* 외부 API마다 응답 특성이 다르다
* AI 서버 호출은 별도 설정이 필요하다
* AI 서버 전용 RestTemplate을 두면 장애 대응과 설정 관리가 쉬워진다

---

## 31. timeout과 외부 서버 장애 대응

AI 서버가 응답하지 않는 상황을 생각해보자.

```text
Spring 백엔드
→ AI 서버 요청
→ AI 서버 응답 없음
```

timeout이 없으면 백엔드 요청이 오래 붙잡힐 수 있다.
그러면 사용자는 계속 기다리게 되고, 서버 자원도 낭비된다.

그래서 외부 API 호출에는 보통 아래 설정이 필요하다.

```text
connect timeout
read timeout
```

### connect timeout

AI 서버와 연결을 맺을 때까지 기다리는 시간이다.

### read timeout

연결 후 응답 데이터를 받을 때까지 기다리는 시간이다.

핵심:

* timeout은 외부 서버 장애가 백엔드 전체 장애로 번지는 것을 막기 위해 필요하다
* AI 서버처럼 응답 시간이 달라질 수 있는 외부 시스템은 별도 timeout 관리가 중요하다
* 실패 시 기본 추천 결과, 빈 리스트, 에러 응답 등 서비스 정책에 맞는 처리가 필요하다

---

## 32. 5단계: AI 서버가 유사도 기반 기관 후보 반환

AI 서버 내부에서는 사용자 조건을 임베딩하고, 기관 벡터들과 비교한다.

개념 예시:

```python
@app.post("/api/v1/recommendations")
def recommend(request: RecommendationRequest):
    user_vector = embedding_model.encode(request.request_text)

    candidates = []

    for institution in institution_vectors:
        score = cosine_similarity(user_vector, institution.vector)

        candidates.append({
            "institutionId": institution.id,
            "similarity": score
        })

    candidates.sort(key=lambda x: x["similarity"], reverse=True)

    return candidates[:10]
```

실제 AI 서버에서는 bge-m3 임베딩과 pgvector 검색을 사용해 유사 기관을 찾는 구조다.

핵심:

* 사용자 조건 → 임베딩
* 기관 정보 → 기존 임베딩
* 유사도 계산
* 유사한 기관 후보 반환

---

## 33. 6단계: 백엔드가 기관 상세 정보 조회

AI 서버가 반환한 결과만으로는 클라이언트 화면을 구성하기 부족할 수 있다.

예를 들어 AI 서버가 다음과 같이 반환할 수 있다.

```json
[
  {
    "institutionId": 12,
    "similarity": 0.91
  },
  {
    "institutionId": 7,
    "similarity": 0.84
  }
]
```

그러면 백엔드는 기관 ID를 기준으로 상세 정보를 다시 조회한다.

예시 코드:

```java
List<Long> institutionIds = aiItems.stream()
        .map(AiRecommendationItem::getInstitutionId)
        .toList();

List<Institution> institutions =
        institutionRepository.findAllById(institutionIds);

Map<Long, Institution> institutionMap = institutions.stream()
        .collect(Collectors.toMap(
                Institution::getId,
                institution -> institution
        ));
```

핵심:

* AI 서버는 추천 후보 계산을 담당한다
* 백엔드는 실제 서비스 화면에 필요한 기관 상세 정보를 조회한다

---

## 34. 7단계: 이미지 URL, 상담 가능 정보 등 후처리

AI 서버 결과는 그대로 클라이언트에 주기 어렵다.

화면에는 보통 다음 정보가 필요하다.

* 기관명
* 주소
* 대표 이미지
* 상담 가능 여부
* 예약 가능 여부
* 기관 설명
* 거리 정보
* 유사도 점수

예시 코드:

```java
private RecommendationResponse toResponse(
        AiRecommendationItem item,
        Institution institution
) {
    String imageUrl = null;

    if (institution.getImageKey() != null) {
        imageUrl = s3Service.createPresignedUrl(institution.getImageKey());
    }

    boolean counselAvailable = institution.isCounselAvailable();
    boolean reservationAvailable = institution.isReservationAvailable();

    return new RecommendationResponse(
            institution.getId(),
            institution.getName(),
            institution.getAddress(),
            imageUrl,
            counselAvailable,
            reservationAvailable,
            item.getSimilarity()
    );
}
```

핵심:

* 추천 결과는 서비스 화면에 맞게 보강되어야 한다
* 백엔드는 AI 결과를 실제 응답 DTO로 변환한다

---

## 35. 8단계: 클라이언트에 추천 결과 응답

최종 응답 예:

```json
[
  {
    "institutionId": 12,
    "institutionName": "햇살 주야간보호센터",
    "address": "서울시 강남구 ...",
    "imageUrl": "https://s3-presigned-url...",
    "counselAvailable": true,
    "reservationAvailable": true,
    "similarity": 0.91
  },
  {
    "institutionId": 7,
    "institutionName": "마음돌봄 데이케어센터",
    "address": "서울시 송파구 ...",
    "imageUrl": "https://s3-presigned-url...",
    "counselAvailable": true,
    "reservationAvailable": false,
    "similarity": 0.84
  }
]
```

핵심:

* 클라이언트는 AI 서버의 원본 결과가 아니라 백엔드가 조립한 최종 결과를 받는다
* 이 구조 덕분에 추천 결과가 상담/예약/상세 화면 흐름과 연결될 수 있다

---

## 36. S3란 무엇인가

S3는 AWS에서 제공하는 파일 저장소다.

이미지, PDF, 첨부파일, 프로필 사진 같은 파일을 서버 디스크에 직접 저장하지 않고 AWS 저장소에 올려두는 방식이다.

어르신 케어서비스에서는 다음과 같은 파일에 사용될 수 있다.

* 기관 대표 이미지
* 기관 소개 이미지
* 상담 관련 첨부파일
* 사용자 프로필 이미지

서버 내부에 파일을 저장하면 서버 재배포나 서버 확장 시 관리가 복잡해진다.

그래서 파일은 애플리케이션 서버가 아니라 S3 같은 외부 저장소에 따로 보관하는 구조가 적합하다.

핵심:

* S3 = 파일 보관 창고
* DB에는 파일 자체가 아니라 파일 key나 경로를 저장
* 실제 파일은 S3에 저장

---

## 37. S3와 DB의 역할 차이

DB에는 파일 자체보다 파일 위치를 저장하는 경우가 많다.

예:

```text
institution_id = 12
name = 햇살 주야간보호센터
image_key = institution/12/main.png
```

S3에는 실제 이미지 파일이 저장된다.

```text
S3 bucket
└── institution/12/main.png
```

역할:

```text
MySQL → 파일이 어디 있는지 기록
S3 → 실제 파일 보관
```

핵심:

* DB는 정보 저장
* S3는 파일 저장

---

## 38. Presigned URL이란 무엇인가

S3 버킷을 전체 공개로 열어두면 보안상 위험하다.

그래서 일반적으로는 버킷을 비공개로 두고, 백엔드가 필요한 순간에만 일정 시간 접근 가능한 임시 URL을 발급한다.

이 임시 URL이 Presigned URL이다.

예:

```text
https://s3.amazonaws.com/bucket/institution/12/main.png?X-Amz-Signature=...
```

핵심:

* Presigned URL = 임시 출입증
* S3 파일을 공개하지 않고 필요한 사용자에게만 접근 허용 가능
* 시간이 지나면 만료되도록 설정할 수 있다

---

## 39. 추천 기능에서 S3가 필요한 이유

AI 서버는 추천 계산을 담당한다.

AI 서버 결과 예:

```json
[
  {
    "institutionId": 12,
    "similarity": 0.91
  }
]
```

하지만 화면에서는 기관 이미지가 필요하다.

추천 응답에서 S3를 사용하는 흐름:

```text
AI 서버가 기관 12번 추천
→ 백엔드가 DB에서 기관 12번 상세 조회
→ image_key 확인
→ S3 Presigned URL 생성
→ imageUrl 포함 응답
```

핵심:

* AI 서버는 이미지 접근 권한을 처리하지 않는다
* 백엔드가 S3 접근 URL을 만들어 응답에 붙인다
* 추천 결과도 기존 파일 보안 정책을 따라야 한다

---

## 40. S3Service 예시

이해용 예시:

```java
@Service
@RequiredArgsConstructor
public class S3Service {

    private final S3Presigner s3Presigner;

    @Value("${aws.s3.bucket}")
    private String bucket;

    public String createPresignedUrl(String imageKey) {
        GetObjectRequest getObjectRequest = GetObjectRequest.builder()
                .bucket(bucket)
                .key(imageKey)
                .build();

        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(10))
                .getObjectRequest(getObjectRequest)
                .build();

        PresignedGetObjectRequest presignedRequest =
                s3Presigner.presignGetObject(presignRequest);

        return presignedRequest.url().toString();
    }
}
```

의미:

* `bucket` = 파일이 들어있는 S3 저장소
* `imageKey` = S3 안에서 파일 위치
* `signatureDuration` = URL 유효 시간
* `presignGetObject` = 임시 접근 URL 생성

핵심:

* S3 파일을 직접 공개하지 않음
* 백엔드가 필요한 순간에 임시 URL을 만들어준다

---

## 41. Redis란 무엇인가

Redis는 메모리 기반의 빠른 key-value 저장소다.

비유하면 다음과 같다.

```text
MySQL = 정식 서류 보관함
Redis = 책상 위 포스트잇
```

Redis는 보통 다음과 같은 데이터에 사용된다.

* 인증 코드
* 임시 회원 정보
* 리프레시 토큰
* 로그인 세션
* 짧은 시간만 필요한 데이터

핵심:

* Redis는 빠르다
* 만료 시간을 설정할 수 있다
* 임시 데이터 저장에 적합하다

---

## 42. Redis가 필요한 이유

예를 들어 이메일 인증 코드를 생각해보자.

```text
사용자 이메일: user@test.com
인증 코드: 482913
유효 시간: 5분
```

이런 데이터는 5분 뒤면 필요 없다.

MySQL에 저장할 수도 있지만, 매번 저장하고 삭제하는 것은 부담스럽다.

Redis를 쓰면 다음처럼 관리할 수 있다.

```text
key = auth:code:user@test.com
value = 482913
TTL = 5분
```

5분이 지나면 자동으로 사라진다.

핵심:

* 짧게 필요한 데이터는 Redis가 적합하다
* TTL로 자동 만료 가능하다
* DB에 불필요한 임시 데이터를 쌓지 않아도 된다

---

## 43. 어르신 케어서비스에서 Redis가 쓰이는 흐름

Redis는 다음 데이터에 활용될 수 있다.

* 인증 코드
* 임시 사용자 정보
* 리프레시 토큰

### 인증 코드 저장

```text
1. 사용자가 이메일/휴대폰 인증 요청
2. 백엔드가 인증 코드 생성
3. Redis에 코드 저장, TTL 설정
4. 사용자가 코드 입력
5. Redis에서 코드 조회
6. 일치하면 인증 성공
```

예:

```text
key = auth:code:user@test.com
value = 482913
TTL = 5분
```

### 임시 사용자 저장

회원가입 과정이 여러 단계인 경우, 아직 정식 회원으로 저장하기 애매한 데이터를 Redis에 잠깐 둘 수 있다.

예:

```text
key = temp:user:user@test.com
value = {name, phone, verified}
TTL = 30분
```

### 리프레시 토큰 저장

JWT 인증에서는 Access Token과 Refresh Token을 나누는 경우가 많다.

```text
Access Token = 짧게 쓰는 출입증
Refresh Token = 새 출입증을 발급받기 위한 긴 출입증
```

Refresh Token을 Redis에 저장하면 다음이 가능하다.

* 빠른 조회
* 만료 시간 관리
* 로그아웃 시 삭제
* 강제 로그아웃 처리

예:

```text
key = refresh:member:12
value = eyJhbGciOiJIUzI1NiJ9...
TTL = 14일
```

핵심:

* Redis는 인증/토큰 관리에 적합하다
* 만료 시간이 있는 데이터와 잘 맞는다

---

## 44. Redis 코드 예시

### 인증 코드 저장

```java
@Service
@RequiredArgsConstructor
public class AuthCodeService {

    private final StringRedisTemplate redisTemplate;

    public void saveAuthCode(String email, String code) {
        String key = "auth:code:" + email;

        redisTemplate.opsForValue().set(
                key,
                code,
                Duration.ofMinutes(5)
        );
    }

    public boolean verifyAuthCode(String email, String inputCode) {
        String key = "auth:code:" + email;

        String savedCode = redisTemplate.opsForValue().get(key);

        return inputCode.equals(savedCode);
    }
}
```

### 리프레시 토큰 저장

```java
@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final StringRedisTemplate redisTemplate;

    public void saveRefreshToken(Long memberId, String refreshToken) {
        String key = "refresh:member:" + memberId;

        redisTemplate.opsForValue().set(
                key,
                refreshToken,
                Duration.ofDays(14)
        );
    }

    public boolean isValidRefreshToken(Long memberId, String refreshToken) {
        String key = "refresh:member:" + memberId;

        String savedToken = redisTemplate.opsForValue().get(key);

        return refreshToken.equals(savedToken);
    }

    public void deleteRefreshToken(Long memberId) {
        String key = "refresh:member:" + memberId;

        redisTemplate.delete(key);
    }
}
```

핵심:

* Redis는 저장, 조회, 삭제가 빠르다
* TTL이 필요한 인증 흐름에 잘 맞는다

---

## 45. S3와 Redis 차이

| 구분    | S3               | Redis        |
| ----- | ---------------- | ------------ |
| 저장 대상 | 파일               | 임시 데이터       |
| 예시    | 이미지, PDF, 첨부파일   | 인증 코드, 토큰    |
| 저장 성격 | 비교적 오래 보관        | 만료 시간 있는 데이터 |
| 접근 방식 | URL/key 기반 파일 접근 | key-value 조회 |
| 강점    | 대용량 파일 저장        | 빠른 조회, TTL   |

핵심:

```text
S3 = 파일 창고
Redis = 빠른 임시 메모장
```

---

## 46. AI 추천 흐름에 S3와 Redis를 붙여보면

AI 추천 자체는 주로 AI 서버와 Spring 백엔드 사이에서 이루어진다.
하지만 실제 서비스에서는 S3와 Redis가 함께 연결된다.

### 추천 응답에서 S3

```text
AI 서버가 추천 기관 ID 반환
→ 백엔드가 기관 상세 조회
→ 기관 이미지 key 확인
→ S3 Presigned URL 생성
→ imageUrl 포함 응답
```

### 인증/사용자 흐름에서 Redis

```text
사용자 로그인 / 인증
→ 인증 코드 또는 refresh token Redis 저장
→ 추천 API 요청 시 인증된 사용자 확인
```

전체 구조:

```text
사용자 인증
→ Redis로 토큰/인증 상태 관리
→ 추천 요청
→ AI 서버 추천
→ DB 기관 상세 조회
→ S3 이미지 URL 생성
→ 클라이언트 응답
```

핵심:

* S3는 추천 결과에 이미지 파일을 붙이는 데 연결된다
* Redis는 사용자 인증/임시 상태 관리에 연결된다

---

## 47. 최종 전체 흐름

어르신 케어서비스의 AI 추천 흐름은 다음처럼 정리할 수 있다.

```text
1. 사용자가 로그인
2. 인증 상태 또는 토큰을 Redis에서 관리
3. 사용자가 어르신 프로필 기반 추천 요청
4. Spring 백엔드가 사용자/어르신 정보를 조회
5. Spring 백엔드가 AI 서버 RecommendationRequest 생성
6. FastAPI AI 서버에 추천 요청
7. AI 서버가 사용자 프로필 텍스트 생성
8. bge-m3가 사용자 조건을 1024차원 벡터로 변환
9. pgvector가 기관 벡터들과 유사도 검색
10. AI 서버가 추천 기관 후보 반환
11. Spring 백엔드가 기관 상세 정보 조회
12. 기관 이미지가 있으면 S3 Presigned URL 생성
13. 상담 가능 여부, 예약 가능 여부 등 후처리
14. 클라이언트에 최종 추천 결과 응답
```

---

## 48. 이번 주 핵심 정리

* 어르신 돌봄 기관 추천은 단순 검색만으로 해결하기 어렵다
* 일반 검색은 조건 기반, AI 추천은 의미 기반이다
* 벡터 임베딩은 문장을 숫자 배열로 바꾸는 방식이다
* 의미가 가깝다는 것은 벡터 유사도 점수가 높다는 뜻이다
* 코사인 유사도 기준에서는 점수가 보통 1을 넘지 않는다
* 조건이 여러 개여도 문장 전체가 하나의 고차원 벡터로 표현된다
* AI 서버는 Spring 내부 클래스가 아니라 별도 FastAPI 서버다
* bge-m3 모델이 텍스트를 1024차원 임베딩으로 변환한다
* bge-m3는 한국어 문장 의미 비교, 자체 서버 운영, 추천 검색 구조에 적합하다
* 후보군으로는 OpenAI Embedding API, Ko-SBERT, multilingual-e5, all-MiniLM 등이 있다
* 모델 선택은 품질뿐 아니라 비용, 보안, 서버 운영, 데이터 특성을 함께 고려해야 한다
* pgvector는 PostgreSQL에서 벡터 유사도 검색을 가능하게 한다
* 기관 임베딩은 기관 정보를 추천용 텍스트로 정리한 뒤 생성된다
* 기관 등록/수정/삭제 시 AI 서버 임베딩도 함께 동기화되어야 한다
* 추천 요청은 member, elderly, additionalText, limit 구조로 전달된다
* AI 서버는 사용자 조건 벡터와 기관 벡터를 비교해 유사한 기관을 반환한다
* Spring 백엔드는 사용자/어르신 프로필 조회, AI 요청 생성, 추천 결과 후처리를 담당한다
* AI 서버 호출은 전용 RestTemplate과 timeout 설정으로 분리해 관리하는 것이 좋다
* S3는 파일 저장소로, 추천 응답에 기관 이미지 URL을 붙일 때 활용된다
* Redis는 인증 코드, 임시 사용자, 리프레시 토큰처럼 만료 시간이 필요한 데이터 관리에 적합하다
* 이 구조는 AI 추천 기능을 단순 모델 호출이 아니라 서비스 흐름에 연결한 사례다

---

AI 기능 기반으로 사용자와 기관 정보를 텍스트로 정규화하고,
bge-m3 임베딩과 pgvector 유사도 검색을 통해 추천 후보를 계산한 뒤,
Spring 백엔드가 이를 실제 서비스 응답으로 조립하는 구조를 이해한 주차였다.